# Documentação de Testes de Integração ou Unitários <br>

## 1. O que são os testes? <br>
São testes projetados para validar a lógica de negócios, a integridade dos dados e a comunicação dos serviços. No projeto **API 6**, foi adotada uma estratégia de cobertura em duas frentes:

1.  **Backend (API):** Garante que o cálculo de safras (*Yield Model*) e as regras de inserção no MongoDB funcionem.
2.  **Frontend (Web):** Garante que os serviços de usuário, autenticação e armazenamento local (*Storage*) se comportem corretamente sem depender da API estar online.

Adotei a estratégia de **Testes Unitários Isolados com Mocks**. Isso verifica cenários reais de uso simulando dependências externas (como o Banco de Dados no back ou a API no front), garantindo velocidade na execução da CI.

## 2. Estrutura dos Testes <br>

### 2.1 Backend <br>

Os testes no projeto estão organizados dentro do diretório tests na raiz da aplicação (apps/api). Eles utilizam o framework Pytest e bibliotecas padrão da indústria Python:

- pytest: O runner principal para execução e descoberta dos testes.

- unittest.mock (MagicMock): Para simular o comportamento do banco de dados MongoDB, permitindo testar o sucesso e falhas de conexão sem precisar de um banco real rodando.

- mongomock: Uma biblioteca auxiliar que simula a estrutura de documentos do Mongo em memória.

### 2.2 Frontend <br>

- Utilizei o framework Jest integrado ao Nx. O foco é testar a lógica de serviço (UserService) e utilitários (Storage) isolando as chamadas HTTP.

- Jest Mocks: Interceptam chamadas do Axios/Service para não bater na rede real.

- Mock de Configuração: Isolamos as variáveis de ambiente (import.meta.env) para evitar erros de sintaxe no ambiente de teste.

### Cada teste segue um fluxo comum: <br>

- Arrange (Configuração): Inicialização dos Fixtures (dados falsos e conexão simulada).

- Act (Ação): Execução da função de negócio (ex: create_yield_event).

- Assert (Validação): Verificação se o método de "salvar" foi chamado corretamente e se os dados retornados estão no formato esperado.

## 2.3. Configuração de Ambiente <br>
Para evitar repetição de código e garantir isolamento, utilizei o recurso de Fixtures do Pytest. Em vez de criar um IntegrationEnvironment complexo, injetei dependências controladas.

O arquivo de testes configura automaticamente o ambiente antes de cada função rodar:

### Exemplo de Fixture (mock_mongo): <br>

```Python

@pytest.fixture
def mock_mongo():
    client = MongoClient()
    db = client['test_db']
    yield db 

```

### Exemplo de Patch (Intercepção): <br>

```Python
@pytest.fixture(autouse=True)
def patch_connect(mock_mongo):
    with patch.object(MongoDB, "connect", return_value=mock_mongo):
        yield

```

### O ambiente padrão inclui: <br>

- Mock do Cliente Mongo: Garante que nenhum dado seja gravado no Atlas (Produção) durante os testes.

- Dados de Exemplo (sample_event): Um dicionário pronto com dados de uma safra (Trigo, 2024, California) para padronizar os testes.

## 3. Como executar os testes <br>

### 3.1. Executando o Backend <br>
O projeto utiliza o gerenciador de dependências Poetry para garantir que os testes rodem em um ambiente isolado e controlado.

### Comando principal <br>
O comando abaixo executa a bateria completa de testes do Backend, gerando um relatório visual no terminal:

```Bash

poetry run pytest

```
Este comando irá:

- Verificar o arquivo pytest.ini para configurações.

- Localizar todos os arquivos que começam com test_ dentro de apps/api.

- Executar as funções de teste e exibir ✅ (Passou) ou ❌ (Falhou).

### Execução de um arquivo específico: <br>
Para validar apenas um módulo (útil durante o desenvolvimento), você pode apontar o arquivo:

```Bash

poetry run pytest tests/test_yield_unit.py

```

### 3.2. Executando o Frontend <br>
Utilizamos o Nx para orquestrar os testes da aplicação web.

Comando:

```Bash

# Na raiz do projeto ou na pasta apps/web
npx nx test web
```

## 4. Convenções e Boas Práticas
### 4.1 Estrutura dos testes
Os testes seguem uma estrutura previsível para facilitar a manutenção:

- Mockagem Explicita: Cada teste deve definir o que espera do banco de dados.

- Exemplo: "Vou testar a busca. Então, finja que o banco retornou uma lista com 1 item".

- Código: yield_collection.find = MagicMock(return_value=[sample_event]).

- Asserções de Chamada: Não verificamos apenas o resultado, verificamos se o banco foi chamado.

- Código: yield_collection.insert_one.assert_called_once().

### 4.2 Configuração de ambiente
Antes de rodar os testes, certifique-se de que:

- O Poetry está instalado e as dependências foram baixadas (poetry install).

- Você está na pasta correta (apps/api).

## 5. Exemplos de Casos de Testes <br> 

### 5.1 Backend: Teste de Criação (Create) <br>
Objetivo: Garantir que o sistema envia o objeto de safra corretamente para a persistência.

Fluxo:

- Inicializa os dados de exemplo (sample_event).

- Configura o Mock do banco para retornar sucesso na inserção.

- Executa a função create_yield_event.

- Valida se o ID retornado corresponde ao valor simulado.

### 5.2 Backend: Teste de Atualização (Update) <br>
Objetivo: Garantir a atualização correta dos dados.

Fluxo:

- Configura o Mock para retornar modified_count = 1.

- Altera um dado no objeto (Ex: Produção).

- Valida se a função executou o método update_one do Mongo com os parâmetros novos.

### 5.3 Frontend: Teste de Serviço de Usuário (UserService) <br>
Objetivo: Garantir que o frontend monta a URL de cadastro corretamente.

Fluxo:

- Mocka a função processPOST para interceptar a requisição.

- Executa createUser(newUser).

- Valida se a função direcionou a chamada para o endpoint /register/ utilizando a URL de Autenticação correta.

### 5.4 Frontend: Teste de Storage (LocalStorage) <br>
Objetivo: Garantir que dados sensíveis são salvos e recuperados corretamente.

Fluxo:

- Salva um objeto JSON no localStorage.

- Executa getLocalStorageData.

- Valida se o retorno é um Objeto devidamente parseado, garantindo a integridade do Token.

## 6. Resolução de Problemas comuns <br>
### Backend - ModuleNotFoundError:

- Causa: Execução do pytest fora do ambiente virtual gerenciado.

- Solução: Utilizar o comando poetry run pytest.

### Frontend - SyntaxError: Cannot use 'import.meta':

- Causa: Incompatibilidade do Jest com variáveis de ambiente nativas do Vite.

- Solução: Garantir que o arquivo src/service/config.ts esteja devidamente mockado no topo do arquivo de teste.

### Frontend - Expected path /user/ but got /users/:

- Causa: Divergência entre a rota da API real e a rota esperada no teste unitário.

- Solução: Ajustar o expect do teste para corresponder exatamente à rota definida na camada de Serviço.

- Testes não encontrados: Verifique se você está na pasta apps/api. O pytest procura os arquivos a partir do diretório atual.
