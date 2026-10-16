# Flash Sale Checkout

MVP didático de checkout e reserva de estoque com dois microsserviços Java 21 / Spring Boot 3.5, Redis e RabbitMQ. O projeto demonstra processamento assíncrono, compensação de estoque, cache-aside, bancos separados por serviço e rastreamento por correlation ID.

Funciona em **Windows** (scripts `.bat`) e em **Linux/macOS** (scripts `.sh`). Redis e RabbitMQ podem ser instalados localmente ou iniciados com o `docker-compose.yml` da raiz.

> Este arquivo é a segunda versão da documentação. O `README.md` original foi mantido sem alterações para comparação. As diferenças estão resumidas em [O que mudou em relação ao README original](#o-que-mudou-em-relação-ao-readme-original).

## Sumário

- [Arquitetura](#arquitetura)
- [Requisitos](#requisitos)
- [Início rápido](#início-rápido)
- [Infraestrutura: Redis e RabbitMQ](#infraestrutura-redis-e-rabbitmq)
- [Compilar e testar](#compilar-e-testar)
- [Executar os serviços](#executar-os-serviços)
- [Demonstração](#demonstração)
- [API](#api)
- [Logs e rastreamento](#logs-e-rastreamento)
- [Scripts disponíveis](#scripts-disponíveis)
- [Solução de problemas](#solução-de-problemas)
- [Preparar arquivo ZIP](#preparar-arquivo-zip)
- [Limites do MVP](#limites-do-mvp)
- [O que mudou em relação ao README original](#o-que-mudou-em-relação-ao-readme-original)

## Arquitetura

```
                    ┌──────────────────────┐
  HTTP :8080  ───►  │   checkout-service   │ ──── Redis :6379 (estoque + cache de pedidos)
                    │  API REST + Thymeleaf│
                    │  H2: checkout-db     │
                    └───┬──────────────▲───┘
         order-created  │              │  order-status-updated
                        │              │  order-compensation
                    ┌───▼──────────────┴───┐
                    │  RabbitMQ :5672      │  painel :15672
                    └───┬──────────────▲───┘
                        │              │
                    ┌───▼──────────────┴───┐
                    │order-processor-service│
                    │  H2: processor-db     │
                    └──────────────────────┘
```

| Módulo | Papel |
|---|---|
| `checkout-service` | API REST, interface Thymeleaf, reserva atômica de estoque no Redis (script Lua), cache de pedidos, banco H2 próprio e consumidor dos eventos de status e compensação. |   |   |
| `order-processor-service` | Consome pedidos, persiste no próprio banco H2 e publica eventos de status ou compensação. |   |   |
| `common-contracts` | Biblioteca com os contratos Java das mensagens RabbitMQ. Não tem regra de domínio nem classe `main`. |   |   |

Filas duráveis:

| Fila | Produtor | Consumidor | Uso |
|---|---|---|---|
| `order-created` | Checkout | Order Processor | Processamento assíncrono do pedido |
| `order-compensation` | Order Processor | Checkout | Devolução do estoque após falha |
| `order-status-updated` | Order Processor ou Checkout | Checkout | Atualização para `CONFIRMED` ou `CANCELLED` no banco e no cache do Checkout |

**Fluxo normal:** o Checkout reserva o estoque no Redis e publica em `order-created`. O Processor persiste o pedido e publica `CONFIRMED` em `order-status-updated`.

**Fluxo de falha:** o Processor persiste `CANCELLED` e publica em `order-compensation`. O Checkout devolve o estoque uma única vez (idempotente) e atualiza o status local.

## Requisitos

| Item | Versão | Observação |
|---|---|---|
| JDK | 21 (64 bits) | Precisa ser **JDK**, com `javac`. Um JRE não basta. |
| Maven | 3.6.3 ou mais recente | Recomenda-se 3.9.x. Na primeira compilação, precisa de acesso ao Maven Central. |
| Redis | 7.x | Local ou via Docker. |
| RabbitMQ | 4.x com o plugin de gerenciamento | Local ou via Docker. |
| Docker + Compose | Opcional | Só para subir Redis e RabbitMQ com o `docker-compose.yml`. |

Portas usadas: `5672` (AMQP), `15672` (painel do RabbitMQ), `6379` (Redis) e `8080` (Checkout).

### Conferindo o JDK

Os scripts recusam qualquer Java que não seja o 21 e também recusam um JRE sem `javac`. Se você tiver várias versões instaladas:

```bash
# Linux/macOS com SDKMAN
sdk list java                 # mostra as versões instaladas
sdk use java 21.0.2-open      # só nesta sessão do terminal
sdk default java 21.0.2-open  # como padrão

java -version && javac -version   # os dois devem mostrar 21
```

No Linux, alguns pacotes da distribuição instalam só o JRE em `/usr/lib/jvm/java-21-openjdk-*`. Se `javac` não existir nessa pasta, instale o pacote `openjdk-21-jdk` ou use o SDKMAN.

No Windows, aponte `JAVA_HOME` para a pasta do JDK 21. Os scripts colocam `%JAVA_HOME%\bin` no início do `PATH`.

## Início rápido

**Linux/macOS, com Docker:**

```bash
docker compose up -d --wait      # Redis + RabbitMQ
./scripts/setup.sh               # verifica JDK/Maven, compila e roda os testes unitários
./scripts/start.sh               # inicia os dois serviços em segundo plano
# abra http://localhost:8080/
```

**Windows, com Redis e RabbitMQ instalados localmente:**

```powershell
./scripts/setup.bat
./scripts/start.bat
```

## Infraestrutura: Redis e RabbitMQ

As conexões já estão configuradas nos `application.yml` dos serviços e não precisam de ajuste para rodar localmente:

| Serviço | Redis | RabbitMQ |
|---|---|---|
| `checkout-service` | `localhost:6379` | `localhost:5672`, `guest`/`guest` |
| `order-processor-service` | não usa | `localhost:5672`, `guest`/`guest` |

### Opção A: Docker Compose (recomendada no Linux/macOS)

O `docker-compose.yml` da raiz sobe:

- `flash-sale-redis`: Redis 7.4, porta 6379.
- `flash-sale-rabbitmq`: RabbitMQ 4.1 com painel de gerenciamento, portas 5672 e 15672, usuário e senha `guest`/`guest`.

Os dois têm healthcheck e volumes nomeados, então os dados ficam guardados entre reinicializações.

```bash
docker compose up -d --wait   # só retorna quando os dois estão saudáveis
docker compose ps             # status
docker compose logs -f        # logs
docker compose down           # para os containers e mantém os dados
docker compose down -v        # para os containers e apaga os volumes
```

### Opção B: instalação local

**RabbitMQ no Windows:**

1. Confira a [matriz de compatibilidade Erlang/RabbitMQ](https://www.rabbitmq.com/docs/which-erlang).
2. Instale o Erlang/OTP 64 bits e o RabbitMQ seguindo as [instruções oficiais para Windows](https://www.rabbitmq.com/docs/install-windows). Os dois instaladores normalmente pedem permissão de administrador.
3. Habilite o painel:

   ```powershell
   rabbitmq-plugins.bat enable rabbitmq_management
   rabbitmqctl.bat status
   ```

4. Abra `http://localhost:15672` e entre com `guest`/`guest`. Mantenha essas credenciais só em `localhost`.

**Redis no Windows com WSL2:**

```powershell
wsl --install -d Ubuntu
```

Depois, no terminal do Ubuntu:

```bash
sudo apt update && sudo apt install redis-server
sudo service redis-server start
redis-cli ping   # deve responder PONG
```

**Linux/macOS sem Docker:** instale o `redis-server` e o `rabbitmq-server` pelo gerenciador de pacotes da plataforma e habilite o plugin `rabbitmq_management`.

### Verificando se estão no ar

```bash
# Linux/macOS
docker exec flash-sale-redis redis-cli ping   # ou só: redis-cli ping
curl -s -u guest:guest http://localhost:15672/api/overview > /dev/null && echo "RabbitMQ OK"
```

```powershell
# Windows
Test-NetConnection localhost -Port 5672
Test-NetConnection localhost -Port 6379
```

## Compilar e testar

O setup verifica o JDK e o Maven e roda `mvn -U clean package` com o `scripts/maven-settings.xml`. Esse arquivo ignora mirrors corporativos e usa direto o Maven Central.

```bash
./scripts/setup.sh     # Linux/macOS
./scripts/setup.bat    # Windows
```

Em redes com proxy, configure o proxy autorizado no Maven da máquina. Não coloque credenciais no repositório.

### Testes unitários

Não precisam de Redis nem de RabbitMQ:

```bash
./scripts/test.sh      # ou ./scripts/test.bat
```

Cobrem a validação do checkout, a falta de estoque, a falha de publicação, o 404 após três tentativas, a confirmação e a falha simulada.

### Teste de integração ponta a ponta

Pré-requisitos:

1. Redis e RabbitMQ no ar.
2. O **`order-processor-service` rodando**. É ele que confirma ou cancela os pedidos; sem ele, o teste espera 15 segundos e falha com `Pedido ... nao chegou a CONFIRMED`.

```bash
docker compose up -d --wait
(cd order-processor-service && java -jar target/order-processor-service-1.0.0-SNAPSHOT.jar) &
./scripts/test.sh integracao      # ou ./scripts/test.bat integracao
```

O teste sobe o Checkout numa porta aleatória e verifica os dois fluxos. No `CONFIRMED`, confere a propagação do correlation ID. No `CANCELLED`, confere se o estoque volta a 10 depois da compensação.

## Executar os serviços

### Com os scripts

```bash
./scripts/start.sh     # Linux/macOS
./scripts/start.bat    # Windows
```

Antes de iniciar, os dois scripts verificam o JDK 21, as portas 5672 e 6379 e a existência dos JARs.

| | `start.bat` (Windows) | `start.sh` (Linux/macOS) |
|---|---|---|
| Como roda | Abre uma janela para cada serviço | Roda os dois em segundo plano com `nohup` |
| Saída do console | Na própria janela | Em `<serviço>/logs/console.out` |
| Como parar | Fechar as janelas | `kill <PID>` (o script mostra os PIDs) |

### Com `java -jar`

Depois do setup, em dois terminais a partir da raiz:

```bash
cd order-processor-service && java -jar target/order-processor-service-1.0.0-SNAPSHOT.jar
cd checkout-service        && java -jar target/checkout-service-1.0.0-SNAPSHOT.jar
```

### Com o Maven (`spring-boot:run`)

Rode **a partir da raiz** e use `-pl` para escolher o serviço. O `-am` compila junto o `common-contracts`, do qual os serviços dependem:

```bash
mvn spring-boot:run -pl checkout-service -am
mvn spring-boot:run -pl order-processor-service -am
```

`mvn spring-boot:run` sem `-pl`, na raiz, inicia só o primeiro serviço do reator (o Checkout) e fica bloqueado nele.

Para rodar `mvn spring-boot:run` de dentro da pasta de um serviço, instale antes os módulos no repositório local, uma vez:

```bash
mvn install -DskipTests
cd checkout-service && mvn spring-boot:run
```

Sem esse passo, o Maven falha com `Could not find artifact br.com.flashsale:common-contracts`.

### Endereços

| O quê | URL |
|---|---|
| Interface do Checkout | http://localhost:8080/ |
| Console H2 do Checkout | http://localhost:8080/h2-console |
| Painel do RabbitMQ | http://localhost:15672/ (`guest`/`guest`) |

O Hibernate cria os bancos H2 em arquivo: `checkout-service/checkout-db.mv.db` e `order-processor-service/processor-db.mv.db`. Cada serviço acessa só o próprio banco.

## Demonstração

### Pela interface

Em `http://localhost:8080/`:

1. Cadastre estoque para `PROD-001`, por exemplo 10 unidades.
2. Envie um pedido com a opção de falha desmarcada. O status vai de `PROCESSING` para `CONFIRMED`.
3. Envie outro pedido com a opção de falha marcada. O pedido chega a `CANCELLED`, a página mostra o motivo e as unidades voltam ao estoque.
4. Use o ID do pedido para consultá-lo e procure o `correlationId` nos logs dos dois serviços.

### Automatizada via HTTP

**Windows (PowerShell):**

```powershell
Set-ExecutionPolicy -Scope Process Bypass
.\scripts\run-demo.ps1
.\scripts\run-demo.ps1 -BaseUrl http://localhost:8080 -ProductId PROD-002
```

**Linux/macOS:** o `run-demo.ps1` ainda não tem versão `.sh`. Os mesmos passos com `curl`:

```bash
curl -s -X PUT localhost:8080/v1/inventory/PROD-001 \
     -H 'Content-Type: application/json' -d '{"quantity": 10}'

curl -s -X POST localhost:8080/v1/orders \
     -H 'Content-Type: application/json' -H "X-Correlation-Id: $(uuidgen)" \
     -d '{"produto_id":"PROD-001","quantidade":2,"cliente_cpf":"50329291076",
          "cliente_nome":"Cliente de demonstracao","simular_falha":false,"motivo_falha":""}'

curl -s localhost:8080/v1/orders/<pedido_id>
```

## API

### Criar pedido: `POST /v1/orders`

Respostas:
- `202 Accepted`: o estoque foi reservado e o RabbitMQ confirmou a publicação.
- `409 Conflict`: o saldo é insuficiente.
- `400 Bad Request`: o corpo é inválido.

```json
{
  "produto_id": "PROD-001",
  "quantidade": 2,
  "cliente_cpf": "50329291076",
  "cliente_nome": "Cliente de demonstracao",
  "simular_falha": false,
  "motivo_falha": "Falha simulada para demonstracao"
}
```

Para testar falha e compensação, envie `"simular_falha": true`. O cabeçalho `X-Correlation-Id` é opcional; se faltar ou for inválido, o Checkout gera um UUID e o devolve na resposta.

### Estoque

- `PUT /v1/inventory/{produto_id}` com `{"quantity": 10}` define o estoque.
- `GET /v1/inventory/{produto_id}` consulta o saldo.

### Consultar pedido: `GET /v1/orders/{pedido_id}`

Consulta primeiro o cache no Redis (TTL de 5 minutos) e, se não encontrar, o banco do Checkout. Faz até três tentativas antes de responder `404 Not Found`.

## Logs e rastreamento

Cada serviço escreve no console e em:

- `checkout-service/logs/checkout-service.log`
- `order-processor-service/logs/order-processor-service.log`

Com o `start.sh`, a saída do console também fica em `<serviço>/logs/console.out`.

Toda linha traz `[correlationId=...]`. Para seguir um pedido pelos dois serviços:

```bash
grep -h "<correlation-id>" */logs/*.log
```

Eventos de negócio registrados: `message_published`, `order_received`, `order_confirmed`, `order_processing_failed`, `stock_compensated` e `order_status_updated`.

## Scripts disponíveis

Todos ficam em `scripts/` e existem nas versões `.bat` (Windows) e `.sh` (Linux/macOS), com o mesmo comportamento, exceto onde a tabela indica.

| Script | O que faz |
|---|---|
| `check-java` | Exige JDK 21, com `java` e `javac`. Respeita o `JAVA_HOME`. |
| `check-prerequisites` | Roda o `check-java` e exige Maven 3.6.3 ou mais recente. |
| `setup` | Verifica os pré-requisitos e roda `mvn -U clean package` com o `maven-settings.xml`. |
| `test` | Testes unitários. Com o argumento `integracao`, roda o teste ponta a ponta. |
| `start` | Verifica Java, portas e JARs e inicia os dois serviços. O `.bat` abre janelas; o `.sh` roda em segundo plano. |
| `clean-for-zip` | Pede confirmação (S/N) e apaga só os artefatos gerados: `target/`, bancos H2 e `logs/`. |
| `run-demo.ps1` | Demonstração automatizada. Só existe em PowerShell. |

## Solução de problemas

| Sintoma | Causa | Solução |
|---|---|---|
| `ERRO: este projeto requer JDK 21` | O `java` do `PATH` é de outra versão. | `sdk use java 21.x` ou ajuste o `JAVA_HOME`. |
| `ERRO: javac nao pertence ao JDK 21 indicado` | O Java 21 encontrado é só um JRE, ou o `javac` do `PATH` é de outra versão. | Aponte o `JAVA_HOME` para um JDK 21 completo. |
| `Unable to find a suitable main class` em `flash-sale-checkout` | Era o `spring-boot:run` tentando iniciar o POM pai. | Corrigido: o pai e o `common-contracts` agora são ignorados pelo `spring-boot:run`. |
| `Could not find artifact ...:common-contracts` | `spring-boot:run` foi executado de dentro do módulo sem `mvn install`. | Use `mvn spring-boot:run -pl <serviço> -am` na raiz. |
| `Connection refused` / `Broker not available` | O RabbitMQ não está no ar. | `docker compose up -d --wait` ou inicie o serviço local. |
| O pedido fica em `PROCESSING` | O `order-processor-service` não está rodando. | Inicie o Processor. |
| `awk: syntax error` com `awk {print $2}` | O programa do awk está sem aspas, e o shell o divide e expande o `$2`. | Use aspas simples: `awk '{print $2}'`. |

## Preparar arquivo ZIP

```bash
./scripts/clean-for-zip.sh    # ou ./scripts/clean-for-zip.bat; confirme com S
```

O script remove só as pastas `target`, os bancos H2 locais e os logs. Não apaga código-fonte, READMEs, scripts, configurações nem o cache global do Maven. Depois, compacte a pasta `flash-sale` inteira.

Na outra máquina: extraia, instale os pré-requisitos, rode o `setup`, suba o Redis e o RabbitMQ e rode o `start`.

## Limites do MVP

- O H2 em arquivo serve para demonstração local; não representa um banco de produção.
- O fluxo usa confirmação de publicação, filas duráveis, retries locais e compensação idempotente. Não implementa Outbox transacional nem entrega exatamente-uma-vez.
- A fila `order-status-updated` é uma extensão explícita ao item 7 do enunciado: mantém os bancos separados sem o Checkout consultar o banco do Processor.
- O teste de integração depende de Redis, RabbitMQ e do `order-processor-service` em execução.
- O `docker-compose.yml` sobe **só a infraestrutura** (Redis e RabbitMQ). Os serviços Java continuam rodando direto na máquina, como dois processos independentes.

## O que mudou em relação ao README original

Mudanças no projeto feitas depois do README original e documentadas aqui:

**Suporte a Linux/macOS**
- Versões `.sh` de todos os scripts `.bat`: `check-java`, `check-prerequisites`, `setup`, `test`, `start` e `clean-for-zip`.
- No `start.sh`, os serviços rodam em segundo plano, e não em janelas separadas como no `start.bat`.

**Infraestrutura com Docker**
- Novo `docker-compose.yml` na raiz com Redis 7.4 e RabbitMQ 4.1 (com painel), usando as portas e credenciais que os `application.yml` já esperavam.
- O README original dizia "Não usa Docker". Agora o Docker é uma opção só para a infraestrutura; a instalação local continua documentada.

**Correções no build (`pom.xml`)**
- `mainClass` declarada no `checkout-service` (`CheckoutApplication`) e no `order-processor-service` (`OrderProcessorApplication`).
- `spring-boot.run.skip=true` no POM pai, herdado pelo `common-contracts`; os dois serviços o sobrescrevem com `false`. Isso corrigiu o `Unable to find a suitable main class` em `flash-sale-checkout`.
- O `order-processor-service` ganhou a dependência `spring-boot-starter-json`. Sem ela, o serviço não iniciava (`NoClassDefFoundError: com/fasterxml/jackson/databind/JavaType`).

**Correções de código**
- `CacheConfiguration` (Checkout) registra o `JavaTimeModule` no serializador do Redis. Antes, `GET /v1/orders/{id}` dava erro 500 (`SerializationException` com o campo `Instant`) assim que o pedido ia para o cache.
- `CheckoutFlowIntegrationTest` ignora respostas que não são 2xx enquanto espera o status mudar. Antes, um 404 durante a espera virava `NullPointerException`.

**Documentação nova**
- Instruções para escolher o JDK 21 quando há várias versões instaladas, e o aviso de que um JRE não basta.
- Como usar `spring-boot:run` com `-pl ... -am`.
- O teste de integração exige o `order-processor-service` rodando.
- Seção de solução de problemas e tabela de scripts.
- Demonstração com `curl` para quem não tem PowerShell.
- Correção: o README original citava o evento de log `order_published`, que não existe no código. O nome certo é `message_published`.

Com essas mudanças, `./scripts/test.sh` (unitários) e `./scripts/test.sh integracao` (ponta a ponta, com Redis e RabbitMQ do `docker-compose.yml` e o Processor rodando) passaram.
