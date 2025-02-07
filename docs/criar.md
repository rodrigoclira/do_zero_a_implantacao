# 📝 Criando uma tarefa

Certo, temos a funcionalidade de listagem de tarefas já funcionando.

- [x] listar as tarefas
- [ ] adicionar tarefa
- [ ] remover tarefa
- [ ] ordenar a listagem por estado
- [ ] finalizar uma tarefa
- [ ] exibir uma tarefa de forma detalhada

Vamos seguir em frente e escrever a funcionalidade de criar uma tarefa.

Continuamos o ciclo do *TDD* e a primeira coisa a se fazer é pensar em um teste que não esteja implementado.

Daqui pra frente sempre que ver ❌ escreva o teste mostrado e em seguida rode os testes que devem falhar.

Logo em seguida deverá aparecer ✅ e o trecho de código que deve ser alterado. Lembre-se de rodar os testes para garantir que estão funcionando.

E não se esqueça que testes vão no arquivo `test_gerenciador.py` e o código em `gerenciador.py`.

## Passo a passo

Se testarmos o recurso de tarefas utilizando o método POST, veremos que teremos como retorno o código de status `405 METHOD NOT ALLOWED`.

Para testar utilize o comando:

`http POST localhost:8000/tarefas`

Isto é porque até agora só implementamos o método get.

Vamos partir disto para escrever nosso primeiro teste. Primeiro teste então verificaremos o recurso `tarefas` utilizando o método `POST`.

O código de status deve ser diferente de 405. O teste pode ser visto abaixo.

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_recurso_tarefas_deve_aceitar_o_verbo_post():
    cliente = TestClient(app)
    resposta = cliente.post("/tarefas")
    assert resposta.status_code != status.HTTP_405_METHOD_NOT_ALLOWED
```

Próxima etapa do ciclo é escrevermos o código suficiente para satisfazer o nosso teste.

O código é simples, vamos criar um novo método `criar` e associá-los ao método `POST`do recurso tarefas.

✅

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
@app.post('/tarefas')
def criar():
    pass
```
Ok, o teste está passando.

Vamos criar uma nova situação onde o nosso código falha.

Na nossa requisição, caso o corpo não tenha um título, deveremos receber o código de status `422 Unprocessable Entity` que significa que a "entidade", que neste caso é a tarefa, foi passada com algum problema.

Vamos transformar isto em um teste.

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_quando_uma_tarefa_e_submetida_deve_possuir_um_titulo():
    cliente = TestClient(app)
    resposta = cliente.post("/tarefas", json={})
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY
```

Agora vamos utilizar o [pydantic](https://pydantic-docs.helpmanual.io/) como desserializador da nossa entrada e validador.

Desseria...o que?

Quando recebemos uma requisição, em seu corpo temos um conteúdo que está no formato json, precisamos ler e entender esta estrutura e transformar em algo que possa manipular no python.

Criaremos então uma Tarefa, que possui um titulo que é uma string. Esta tarefa é baseada em um modelo da biblioteca pydantic.

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
from pydantic import BaseModel

# ...

class Tarefa(BaseModel):
    titulo: str
```

Adicionamos então ao método criar uma tarefa e isto é suficiente para ele saber que ao ser acessado via post, deve conter em seu corpo uma tarefa com um título.

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
@app.post('/tarefas')
def criar(tarefa: Tarefa):
    pass
```

E o resultado final que faz os testes passarem é:

✅

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
from fastapi import FastAPI
from pydantic import BaseModel


app = FastAPI()


class Tarefa(BaseModel):
    titulo: str


TAREFAS = []


@app.get('/tarefas')
def listar():
    return TAREFAS


@app.post('/tarefas')
def criar(tarefa: Tarefa):
    pass
```

E o ciclo continua, temos uma restrição no titulo que é "deve possuir entre 3 e 50 caracteres", vamos testar isto.

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_titulo_da_tarefa_deve_conter_entre_3_e_50_caracteres():
    cliente = TestClient(app)
    resposta = cliente.post("/tarefas", json={"titulo": 2 * "*"})
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY
    resposta = cliente.post("/tarefas", json={"titulo": 51 * "*"})
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY
```

Para resolver esta validação substituiremos o tipo `str` da nossa tarefa por `constr`, que em inglês quer dizer "_constrained str_", e em bom português "string com restrições".

Definimos então `min_length`(comprimento mínimo) como 3 e `max_length`(comprimento máximo) como 50.

✅

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
from pydantic import BaseModel, constr



class Tarefa(BaseModel):
    titulo: constr(min_length=3, max_length=50)
```

Testes passando, vamos continuar a construir nossa tarefa.

Além de titulo, nossa tarefa deve possuir uma descrição.

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_quando_uma_tarefa_e_submetida_deve_possuir_uma_descricao():
    cliente = TestClient(app)
    resposta = cliente.post("/tarefas", json={"titulo": "titulo"})
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY
```

Adicionamos a nossa tarefa o campo descrição.

✅

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
class Tarefa(BaseModel):
    titulo: constr(min_length=3, max_length=50)
    descricao: str
```

Mas a descrição só pode ter 140 caracteres.

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_descricao_da_tarefa_pode_conter_no_maximo_140_caracteres():
    cliente = TestClient(app)
    resposta = cliente.post("/tarefas", json={"titulo": "titulo", "descricao": "*" * 141})
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY
```

Assim como o título, vamos mudar de `str` para `constr` e adicionar a restrição no comprimento do texto.

✅

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
class Tarefa(BaseModel):
    titulo: constr(min_length=3, max_length=50)
    descricao: constr(max_length=140)
```

Outra coisa é ao pedir a criação da tarefa, a mesma deve ser retornada como resposta.

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_quando_criar_uma_tarefa_a_mesma_deve_ser_retornada():
    cliente = TestClient(app)
    tarefa_esperada = {"titulo": "titulo", "descricao": "descricao"}
    resposta = cliente.post("/tarefas", json=tarefa_esperada)
    tarefa_criada = resposta.json()
    assert tarefa_criada["titulo"] == tarefa_esperada["titulo"]
    assert tarefa_criada["descricao"] == tarefa_esperada["descricao"]
    TAREFAS.clear()
```

🤔 E se eu retornar a tarefa?

✅

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
@app.post('/tarefas')
def criar(tarefa: Tarefa):
    return tarefa
```

😅 Esta foi simples.

Outra coisa que precisamos verificar é que cada tarefa deve possuir um identificador único.

Para checar isto vamos adicionar duas tarefas e seus `ids`retornados devem ser diferentes.

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_quando_criar_uma_tarefa_seu_id_deve_ser_unico():
    cliente = TestClient(app)
    tarefa1 = {"titulo": "titulo1", "descricao": "descricao1"}
    tarefa2 = {"titulo": "titulo2", "descricao": "descricao1"}
    resposta1 = cliente.post("/tarefas", json=tarefa1)
    resposta2 = cliente.post("/tarefas", json=tarefa2)
    assert resposta1.json()["id"] != resposta2.json()["id"]
    TAREFAS.clear()
```

Como o `id` é uma coisa que só deve aparecer na resposta, vamos a algumas mudanças.

A primeira é que renomearemos a nossa `Tarefa`para `TarefaEntrada`e criaremos uma segunda estrutura Tarefa que é baseada na entrada, porém possui também um id.

Para torna-lo único, o faremos do tipo [uuid](https://pt.wikipedia.org/wiki/Identificador_%C3%BAnico_universal), que é um identificador universalmente único.

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
from uuid import UUID, uuid4

# ...

class TarefaEntrada(BaseModel):
    titulo: constr(min_length=3, max_length=50)
    descricao: constr(max_length=140)


class Tarefa(TarefaEntrada):
    id: UUID
```

Depois vamos no método criar e transformar nossa tarefa de entrada em um dicionário, em seguida, adicionamos um id único gerado pelo python.

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
@app.post('/tarefas')
def criar(tarefa: TarefaEntrada):
    nova_tarefa = tarefa.dict()
    nova_tarefa.update({"id": uuid4()})
    return nova_tarefa
```

Outro detalhe é avisar ao nosso método post que utilize nossa nova estrutura para gerar a saída no formato json.

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
@app.post('/tarefas', response_model=Tarefa)
def criar(tarefa: TarefaEntrada):
```

✅

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
from uuid import UUID, uuid4


class TarefaEntrada(BaseModel):
    titulo: constr(min_length=3, max_length=50)
    descricao: constr(max_length=140)


class Tarefa(TarefaEntrada):
    id: UUID


@app.post('/tarefas', response_model=Tarefa)
def criar(tarefa: TarefaEntrada):
    nova_tarefa = tarefa.dict()
    nova_tarefa.update({"id": uuid4()})
    return nova_tarefa
```

Certo, testes passando novamente. Ainda temos alguma coisa pra verificar?

Sim! Nossa tarefa também deve possuir um estado que por padrão será "não finalizado".

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_quando_criar_uma_tarefa_seu_estado_padrao_e_nao_finalizado():
    cliente = TestClient(app)
    tarefa = {"titulo": "titulo", "descricao": "descricao"}
    resposta = cliente.post("/tarefas", json=tarefa)
    assert resposta.json()["estado"] == "não finalizado"
    TAREFAS.clear()
```

Como temos apenas dois estados possíveis (finalizado, não finalizado) para uma tarefa, vamos utilizar uma estrutura do Python que é bastante útil para estes momentos.

O Enum, é uma estrutura que define valores limitados a algo.

Um exemplo poderia ser os estados do nosso país, tamanhos de roupa, cores.

Mas por que?

Vamos pegar como exemplo o tamanho de roupa. Inicialmente nosso sistema possuia, "Pequena", "Média", etc. De repente por uma questão de economia de espaço, estes valores modificam para "p", "m".

E agora? vamos ter que ir em cada lugar do sistema que utiliza os valores e realizar a substituição. Mas e se eu esquecer e utilizar o antigo.

Então ao invés de utilizarmos ``

Adicionamos estado a estrutura TarefaEntrada, e seu tipo é `EstadosPossiveis`.

Um valor padrão será `EstadosPossiveis.nao_finalizado`.

Você deve estar se perguntando por que `EstadosPossiveis.nao_finalizado`e não a string direto. É justamente para evitar o problema citado acima de substituição.

✅

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
from enum import Enum


class EstadosPossiveis(str, Enum):
    finalizado = "finalizado"
    nao_finalizado = "não finalizado"


class TarefaEntrada(BaseModel):
    titulo: constr(min_length=3, max_length=50)
    descricao: constr(max_length=140)
    estado: EstadosPossiveis = EstadosPossiveis.nao_finalizado
```

Quase tudo certo, porém o código de status quando algo é criado deve ser `201 Created`.

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_quando_criar_uma_tarefa_codigo_de_status_retornado_deve_ser_201():
    cliente = TestClient(app)
    tarefa = {"titulo": "titulo", "descricao": "descricao"}
    resposta = cliente.post("/tarefas", json=tarefa)
    assert resposta.status_code == status.HTTP_201_CREATED
    TAREFAS.clear()
```

Modifique o método para retornar 201 quando for bem sucedido.

✅

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
from fastapi import FastAPI, status

# ...

@app.post('/tarefas', response_model=Tarefa, status_code=status.HTTP_201_CREATED)
```

A última coisa é que no momento não estamos guardando a nova tarefa.

❌

```{.py3 title="tests/test_gerenciador.py"}
def test_quando_criar_uma_tarefa_esta_deve_ser_persistida():
    cliente = TestClient(app)
    tarefa = {"titulo": "titulo", "descricao": "descricao"}
    cliente.post("/tarefas", json=tarefa)
    assert len(TAREFAS) == 1
    TAREFAS.clear()

```

✅

```{.py3 hl_lines="5" title="gerenciador.py"}
@app.post('/tarefas', response_model=Tarefa, status_code=status.HTTP_201_CREATED)
def criar(tarefa: TarefaEntrada):
    nova_tarefa = tarefa.dict()
    nova_tarefa.update({"id": uuid4()})
    TAREFAS.append(nova_tarefa)
    return nova_tarefa
```

🎉  Nossos testes passaram! Agora temos as funcionalidades de criação e listagem de tarefas.

- [x] listar as tarefas
- [x] adicionar tarefa

No fim os testes ficam similar a:

```{.py3 title="tests/test_gerenciador.py"}
from fastapi import status
from fastapi.testclient import TestClient

from gerencia´dor_tarefas.gerenciador import TAREFAS, app


def test_quando_listar_tarefas_devo_ter_como_retorno_codigo_de_status_200():
    cliente = TestClient(app)
    resposta = cliente.get("/tarefas")
    assert resposta.status_code == status.HTTP_200_OK


def test_quando_listar_tarefas_formato_de_retorno_deve_ser_json():
    cliente = TestClient(app)
    resposta = cliente.get("/tarefas")
    assert resposta.headers["Content-Type"] == "application/json"


def test_quando_listar_tarefas_retorno_deve_ser_uma_lista():
    cliente = TestClient(app)
    resposta = cliente.get("/tarefas")
    assert isinstance(resposta.json(), list)


def test_quando_listar_tarefas_a_tarefa_retornada_deve_possuir_id():
    TAREFAS.append(
        {
            "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
            "titulo": "titulo 1",
            "descricao": "descricao 1",
            "estado": "finalizado",
        }
    )
    cliente = TestClient(app)
    resposta = cliente.get("/tarefas")
    assert "id" in resposta.json().pop()
    TAREFAS.clear()


def test_quando_listar_tarefas_a_tarefa_retornada_deve_possuir_titulo():
    TAREFAS.append(
        {
            "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
            "titulo": "titulo 1",
            "descricao": "descricao 1",
            "estado": "finalizado",
        }
    )
    cliente = TestClient(app)
    resposta = cliente.get("/tarefas")
    assert "titulo" in resposta.json().pop()
    TAREFAS.clear()


def test_quando_listar_tarefas_a_tarefa_retornada_deve_possuir_descricao():
    TAREFAS.append(
        {
            "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
            "titulo": "titulo 1",
            "descricao": "descricao 1",
            "estado": "finalizado",
        }
    )
    cliente = TestClient(app)
    resposta = cliente.get("/tarefas")
    assert "descricao" in resposta.json().pop()
    TAREFAS.clear()


def test_quando_listar_tarefas_a_tarefa_retornada_deve_possuir_um_estado():
    TAREFAS.append(
        {
            "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
            "titulo": "titulo 1",
            "descricao": "descricao 1",
            "estado": "finalizado",
        }
    )
    cliente = TestClient(app)
    resposta = cliente.get("/tarefas")
    assert "estado" in resposta.json().pop()
    TAREFAS.clear()


def test_recurso_tarefas_deve_aceitar_o_verbo_post():
    cliente = TestClient(app)
    resposta = cliente.post("/tarefas")
    assert resposta.status_code != status.HTTP_405_METHOD_NOT_ALLOWED


def test_quando_uma_tarefa_e_submetida_deve_possuir_um_titulo():
    cliente = TestClient(app)
    resposta = cliente.post("/tarefas", json={})
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY


def test_titulo_da_tarefa_deve_conter_entre_3_e_50_caracteres():
    cliente = TestClient(app)
    resposta = cliente.post("/tarefas", json={"titulo": 2 * "*"})
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY
    resposta = cliente.post("/tarefas", json={"titulo": 51 * "*"})
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY


def test_quando_uma_tarefa_e_submetida_deve_possuir_uma_descricao():
    cliente = TestClient(app)
    resposta = cliente.post("/tarefas", json={"titulo": "titulo"})
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY


def test_descricao_da_tarefa_pode_conter_no_maximo_140_caracteres():
    cliente = TestClient(app)
    resposta = cliente.post(
        "/tarefas", json={"titulo": "titulo", "descricao": "*" * 141}
    )
    assert resposta.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY


def test_quando_criar_uma_tarefa_a_mesma_deve_ser_retornada():
    cliente = TestClient(app)
    tarefa_esperada = {"titulo": "titulo", "descricao": "descricao"}
    resposta = cliente.post("/tarefas", json=tarefa_esperada)
    tarefa_criada = resposta.json()
    assert tarefa_criada["titulo"] == tarefa_esperada["titulo"]
    assert tarefa_criada["descricao"] == tarefa_esperada["descricao"]
    TAREFAS.clear()


def test_quando_criar_uma_tarefa_seu_id_deve_ser_unico():
    cliente = TestClient(app)
    tarefa1 = {"titulo": "titulo1", "descricao": "descricao1"}
    tarefa2 = {"titulo": "titulo2", "descricao": "descricao1"}
    resposta1 = cliente.post("/tarefas", json=tarefa1)
    resposta2 = cliente.post("/tarefas", json=tarefa2)
    assert resposta1.json()["id"] != resposta2.json()["id"]
    TAREFAS.clear()


def test_quando_criar_uma_tarefa_seu_estado_padrao_e_nao_finalizado():
    cliente = TestClient(app)
    tarefa = {"titulo": "titulo", "descricao": "descricao"}
    resposta = cliente.post("/tarefas", json=tarefa)
    assert resposta.json()["estado"] == "não finalizado"
    TAREFAS.clear()


def test_quando_criar_uma_tarefa_codigo_de_status_retornado_deve_ser_201():
    cliente = TestClient(app)
    tarefa = {"titulo": "titulo", "descricao": "descricao"}
    resposta = cliente.post("/tarefas", json=tarefa)
    assert resposta.status_code == status.HTTP_201_CREATED
    TAREFAS.clear()


def test_quando_criar_uma_tarefa_esta_deve_ser_persistida():
    cliente = TestClient(app)
    tarefa = {"titulo": "titulo", "descricao": "descricao"}
    cliente.post("/tarefas", json=tarefa)
    assert len(TAREFAS) == 1
    TAREFAS.clear()

```

E o código:

```{.py3 title="gerenciador_tarefas/gerenciador.py"}
from enum import Enum
from uuid import UUID, uuid4

from fastapi import FastAPI, status
from pydantic import BaseModel, constr

app = FastAPI()


class EstadosPossiveis(str, Enum):
    finalizado = "finalizado"
    nao_finalizado = "não finalizado"


class TarefaEntrada(BaseModel):
    titulo: constr(min_length=3, max_length=50)
    descricao: constr(max_length=140)
    estado: EstadosPossiveis = EstadosPossiveis.nao_finalizado


class Tarefa(TarefaEntrada):
    id: UUID


TAREFAS = []


@app.get("/tarefas")
def listar():
    return TAREFAS


@app.post(
    "/tarefas", response_model=Tarefa, status_code=status.HTTP_201_CREATED
)
def criar(tarefa: TarefaEntrada):
    nova_tarefa = tarefa.dict()
    nova_tarefa.update({"id": uuid4()})
    TAREFAS.append(nova_tarefa)
    return nova_tarefa

```

## 🔧 Testando manualmente

Para testar nossa aplicação manualmente, precisamos colocar nossa aplicação no ar.

Relembrando o comando para isto é `uvicorn --reload gerenciador_tarefas.gerenciador:app`.

Experimente adicionar algumas tarefas utilizando o `httpie`.

![implementação da criação de tarefas](imgs/criar.png "implementação da criação de tarefas")

`http localhost:8000 titulo="titulo" descricao="uma descrição qualquer"`

Lembrando que sempre temos a opção de verificar os recursos através da [documentação](http://localhost:8000/docs) gerada automaticamente.

![documentação com métodos implementados](imgs/get-e-post.png "documentação com métodos implementados")

## Salvando a versão atual do código

Primeiro passo é checar o que foi feito até agora:

```bash
$ git status
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   gerenciador_tarefas/gerenciador.py
        modified:   tests/test_gerenciador.py

no changes added to commit (use "git add" and/or "git commit -a")
```

🎨 Para garantir consistência no estilo do nosso  código, vamos rodar os nossos linters.

```bash
$ python -m isort .
$ python -m black .
$ python -m flake8 --exclude=.venv
```

Caso algum erro seja encontrado, será necessário manualmente corrigí-lo.

Vamos adicionar as alterações nos arquivos.

```
git add gerenciador_tarefas/gerenciador.py tests/test_gerenciador.py
```

💾 Agora vamos marcar esta versão como salva.

```
git commit -m "Adiciona funcionalidade de criar tarefas"
```

🔧 Por fim envie ao GitHub a versão atualizada do projeto.

```
git push
```

☁️ E coloque no ar a nova versão.

```
git push heroku main
```

🎉  Bom trabalho! Vamos então nos desafiar agora nos proximos pasos!
