# React + Fetch — Consumindo um Backend com JWT

Este material ensina, passo a passo, como consumir uma API usando **React** e a função nativa `fetch()` do JavaScript.

O projeto usado como exemplo possui:

- cadastro de usuário;
- login;
- autenticação com JSON Web Token (JWT);
- listagem de posts;
- criação de posts;
- edição dos próprios posts;
- exclusão dos próprios posts.

---

# 1. O que é `fetch`?

`fetch()` é uma função nativa do JavaScript usada para fazer requisições HTTP.

Ela permite que o frontend converse com um backend.

Por exemplo:

```js
fetch("http://localhost:3000/posts")
```

Nesse caso, o navegador faz uma requisição para:

```text
GET http://localhost:3000/posts
```

Como não informamos nenhum método, o `fetch` usa `GET` automaticamente.

---

# 2. Frontend e backend são aplicações diferentes

Neste projeto teremos duas aplicações rodando ao mesmo tempo.

Backend:

```text
http://localhost:3000
```

Frontend React:

```text
http://localhost:5173
```

O frontend não acessa diretamente o banco de dados.

O fluxo é:

```text
React
  ↓
fetch()
  ↓
Backend
  ↓
Service
  ↓
TypeORM
  ↓
MySQL
```

Depois o caminho volta:

```text
MySQL
  ↓
Backend
  ↓
JSON
  ↓
fetch()
  ↓
React
```

---

# 3. Criando o projeto React

No terminal:

```bash
npm create vite@latest frontend -- --template react
```

Entre na pasta:

```bash
cd frontend
```

Instale as dependências:

```bash
npm install
```

Neste projeto também usamos ícones:

```bash
npm install lucide-react
```

Depois:

```bash
npm run dev
```

---

# 4. Organização do projeto

A estrutura utilizada é:

```text
src/
│
├── components/
│   ├── AuthForm.jsx
│   ├── PostForm.jsx
│   └── PostList.jsx
│
├── services/
│   └── api.js
│
├── App.jsx
├── main.jsx
└── styles.css
```

O arquivo mais importante para entender `fetch` será:

```text
src/services/api.js
```

Ele concentra as funções que conversam com o backend.

---

# 5. Nosso endereço do backend

No começo de `api.js`:

```js
const API_URL = "http://localhost:3000"
```

Assim não precisamos repetir esse endereço inteiro em toda requisição.

Em vez de:

```js
fetch("http://localhost:3000/posts")
```

podemos fazer:

```js
fetch(`${API_URL}/posts`)
```

---

# 6. Primeiro `fetch`: GET

Vamos buscar todos os posts.

```js
export async function listarPosts() {

    const response = await fetch(`${API_URL}/posts`)

    const result = await response.json()

    return result
}
```

Vamos entender cada linha.

---

## 6.1 `async`

A função foi declarada assim:

```js
async function listarPosts()
```

Isso significa que dentro dela podemos usar:

```js
await
```

Requisições HTTP não são instantâneas.

O navegador precisa esperar o backend responder.

---

# 7. `await fetch()`

```js
const response = await fetch(`${API_URL}/posts`)
```

O `await` manda o JavaScript esperar a resposta da requisição.

A variável:

```js
response
```

ainda não contém diretamente os posts.

Ela contém um objeto `Response`.

---

# 8. Transformando a resposta em JSON

O backend retorna JSON.

Então usamos:

```js
const result = await response.json()
```

Agora `result` contém os dados enviados pelo backend.

Exemplo:

```json
[
    {
        "id": 1,
        "title": "Meu primeiro post",
        "content": "Olá!",
        "user": {
            "id": 3,
            "name": "Leonardo"
        }
    }
]
```

---

# 9. Verificando se houve erro

Um ponto muito importante:

`fetch()` não lança erro automaticamente apenas porque o backend respondeu `400`, `401`, `403`, `404` ou `500`.

Por isso fazemos:

```js
if (!response.ok) {
    throw new Error(result.message || "Erro ao carregar posts.")
}
```

`response.ok` será `true` normalmente quando a resposta estiver entre `200` e `299`.

Exemplos:

```text
200 OK
201 Created
204 No Content
```

Se vier:

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
500 Internal Server Error
```

então:

```js
response.ok
```

será `false`.

---

# 10. GET completo

Nossa função fica:

```js
export async function listarPosts() {

    const response = await fetch(`${API_URL}/posts`)

    const result = await response.json()

    if (!response.ok) {
        throw new Error(result.message || "Erro ao carregar posts.")
    }

    return result
}
```

---

# 11. Usando essa função no React

No `App.jsx`:

```js
async function carregarPosts() {

    try {

        const data = await listarPosts()

        setPosts(data)

    } catch (error) {

        setErro(error.message)
    }
}
```

Aqui temos:

```js
const data = await listarPosts()
```

Quando a requisição terminar, os posts serão colocados no estado:

```js
setPosts(data)
```

---

# 12. Carregando quando a página abre

Usamos:

```js
useEffect(() => {

    carregarPosts()

}, [])
```

O array vazio:

```js
[]
```

faz esse efeito executar quando o componente é carregado.

Então:

```text
Página abriu
    ↓
useEffect
    ↓
carregarPosts()
    ↓
fetch()
    ↓
GET /posts
    ↓
setPosts()
    ↓
React atualiza a tela
```

---

# 13. POST

Agora precisamos enviar dados.

Por exemplo, cadastrar um usuário.

A requisição será:

```text
POST /users
```

Código:

```js
export async function cadastrarUsuario(data) {

    const response = await fetch(`${API_URL}/users`, {

        method: "POST",

        headers: {
            "Content-Type": "application/json"
        },

        body: JSON.stringify(data)
    })

    const result = await response.json()

    if (!response.ok) {
        throw new Error(result.message)
    }

    return result
}
```

---

# 14. `method`

Quando não informamos nada, o `fetch` usa `GET`.

Para cadastrar precisamos dizer:

```js
method: "POST"
```

Outros exemplos:

```js
method: "PUT"
```

```js
method: "DELETE"
```

---

# 15. `headers`

Estamos enviando JSON.

Então precisamos informar ao backend:

```js
headers: {
    "Content-Type": "application/json"
}
```

Isso equivale a dizer:

```text
O conteúdo que estou enviando está no formato JSON.
```

---

# 16. `body`

O corpo da requisição contém os dados enviados.

Exemplo:

```js
const usuario = {
    name: "Ana",
    email: "ana@email.com",
    password: "123456"
}
```

Mas o `body` do `fetch` precisa ser texto.

Então usamos:

```js
body: JSON.stringify(usuario)
```

O JavaScript converte:

```js
{
    name: "Ana",
    email: "ana@email.com"
}
```

para algo semelhante a:

```json
{
    "name": "Ana",
    "email": "ana@email.com"
}
```

---

# 17. Login

O login também é um `POST`.

```js
export async function fazerLogin(data) {

    const response = await fetch(`${API_URL}/auth/login`, {

        method: "POST",

        headers: {
            "Content-Type": "application/json"
        },

        body: JSON.stringify(data)
    })

    const result = await response.json()

    if (!response.ok) {
        throw new Error(result.message || "Erro ao fazer login.")
    }

    return result
}
```

Enviamos:

```json
{
    "email": "ana@email.com",
    "password": "123456"
}
```

O backend responde algo semelhante a:

```json
{
    "user": {
        "id": 1,
        "name": "Ana",
        "email": "ana@email.com"
    },
    "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

---

# 18. O que fazer com o token?

Depois do login:

```js
const result = await fazerLogin({
    email,
    password
})
```

Temos:

```js
result.user
```

e:

```js
result.token
```

Podemos guardar o token:

```js
localStorage.setItem("token", result.token)
```

---

# 19. `localStorage`

O `localStorage` permite guardar pequenos valores no navegador.

Salvar:

```js
localStorage.setItem("token", token)
```

Ler:

```js
const token = localStorage.getItem("token")
```

Remover:

```js
localStorage.removeItem("token")
```

Como `localStorage` guarda texto, objetos precisam ser convertidos.

Salvar:

```js
localStorage.setItem(
    "usuario",
    JSON.stringify(usuario)
)
```

Ler:

```js
const usuario = JSON.parse(
    localStorage.getItem("usuario")
)
```

---

# 20. Criando um post autenticado

A rota:

```text
POST /posts
```

exige JWT.

Então precisamos enviar o token no cabeçalho da requisição.

```js
export async function criarPost(data, token) {

    const response = await fetch(`${API_URL}/posts`, {

        method: "POST",

        headers: {

            "Content-Type": "application/json",

            "Authorization": `Bearer ${token}`
        },

        body: JSON.stringify(data)
    })

    const result = await response.json()

    if (!response.ok) {
        throw new Error(result.message || "Erro ao criar post.")
    }

    return result
}
```

---

# 21. Authorization Bearer

Esta parte:

```js
"Authorization": `Bearer ${token}`
```

gera um cabeçalho semelhante a:

```text
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

É exatamente o formato que o middleware de autenticação do backend espera.

---

# 22. Por que não enviamos `userId`?

Ao criar um post enviamos apenas:

```js
{
    title,
    content
}
```

Não fazemos:

```js
{
    title,
    content,
    userId
}
```

O backend identifica o usuário pelo JWT.

Isso evita que alguém faça:

```js
{
    title: "Post falso",
    content: "...",
    userId: 15
}
```

e tente criar um post em nome de outra pessoa.

O fluxo correto é:

```text
Frontend envia JWT
        ↓
authMiddleware valida
        ↓
JWT informa o id do usuário
        ↓
Backend associa o post ao usuário correto
```

---

# 23. PUT

Para atualizar:

```text
PUT /posts/5
```

Código:

```js
export async function atualizarPost(id, data, token) {

    const response = await fetch(`${API_URL}/posts/${id}`, {

        method: "PUT",

        headers: {
            "Content-Type": "application/json",
            "Authorization": `Bearer ${token}`
        },

        body: JSON.stringify(data)
    })

    const result = await response.json()

    if (!response.ok) {
        throw new Error(result.message)
    }

    return result
}
```

Observe:

```js
`${API_URL}/posts/${id}`
```

Se:

```js
id = 5
```

a URL será:

```text
http://localhost:3000/posts/5
```

---

# 24. O backend ainda verifica o dono

Mesmo que o frontend esconda o botão de editar de outros usuários, isso não é segurança suficiente.

Um usuário poderia tentar criar manualmente:

```text
PUT /posts/5
```

Por isso o backend verifica:

```text
O usuário do JWT é o dono do post?
```

Se não for:

```text
403 Forbidden
```

O frontend ajuda a experiência do usuário.

O backend garante a segurança.

---

# 25. DELETE

Excluir é um pouco diferente.

```js
export async function excluirPost(id, token) {

    const response = await fetch(`${API_URL}/posts/${id}`, {

        method: "DELETE",

        headers: {
            "Authorization": `Bearer ${token}`
        }
    })

    if (!response.ok) {

        const result = await response.json()

        throw new Error(result.message)
    }
}
```

---

# 26. Por que não usamos `response.json()` sempre no DELETE?

O backend responde:

```text
204 No Content
```

`204` significa que a requisição funcionou, mas a resposta não possui conteúdo.

Por isso isto pode dar problema:

```js
const result = await response.json()
```

Não há JSON para converter.

Então fazemos:

```js
if (!response.ok) {

    const result = await response.json()

    throw new Error(result.message)
}
```

Só tentamos ler o JSON quando houve erro.

---

# 27. Resumo dos métodos

| Ação | Método HTTP | Rota |
|---|---|---|
| Listar posts | GET | `/posts` |
| Buscar post | GET | `/posts/:id` |
| Cadastrar usuário | POST | `/users` |
| Login | POST | `/auth/login` |
| Criar post | POST | `/posts` |
| Atualizar post | PUT | `/posts/:id` |
| Excluir post | DELETE | `/posts/:id` |

---

# 28. Resumo do `fetch`

GET:

```js
fetch(url)
```

POST:

```js
fetch(url, {
    method: "POST",
    headers: {
        "Content-Type": "application/json"
    },
    body: JSON.stringify(dados)
})
```

Com JWT:

```js
fetch(url, {
    headers: {
        "Authorization": `Bearer ${token}`
    }
})
```

POST com JWT:

```js
fetch(url, {
    method: "POST",

    headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${token}`
    },

    body: JSON.stringify(dados)
})
```

---

# 29. Fluxo completo do login

```text
Usuário digita email e senha
        ↓
React
        ↓
fetch()
        ↓
POST /auth/login
        ↓
Backend verifica usuário
        ↓
Backend verifica senha
        ↓
Backend gera JWT
        ↓
Frontend recebe JWT
        ↓
localStorage
```

Depois:

```text
Usuário cria post
        ↓
React pega JWT do estado/localStorage
        ↓
fetch()
        ↓
Authorization: Bearer TOKEN
        ↓
Backend valida JWT
        ↓
Backend identifica usuário
        ↓
Post é criado
```

---

# 30. `try` e `catch`

Como uma requisição pode falhar, usamos:

```js
try {

    const data = await listarPosts()

    setPosts(data)

} catch (error) {

    console.log(error.message)
}
```

O código dentro de:

```js
try
```

é executado normalmente.

Se acontecer um erro lançado com:

```js
throw new Error(...)
```

o JavaScript vai para:

```js
catch
```

---

# 31. Um padrão muito usado

Ao trabalhar com `fetch`, você verá muito este padrão:

```js
async function buscarDados() {

    try {

        const response = await fetch(url)

        const data = await response.json()

        if (!response.ok) {
            throw new Error(data.message)
        }

        return data

    } catch (error) {

        console.log(error.message)
    }
}
```

Entender esse padrão já permite consumir a maioria das APIs REST simples.

---

# 32. Exercícios

## Exercício 1

Crie uma função:

```js
buscarPostPorId(id)
```

que faça:

```text
GET /posts/:id
```

---

## Exercício 2

Crie um botão:

```text
Ver detalhes
```

que carregue apenas um post usando:

```text
GET /posts/:id
```

---

## Exercício 3

Mostre na tela:

```text
Carregando...
```

enquanto os posts estão sendo buscados.

---

## Exercício 4

Crie um estado:

```js
const [erro, setErro] = useState("")
```

e mostre a mensagem enviada pelo backend quando uma requisição falhar.

---

## Exercício 5

Depois de criar um post, chame novamente:

```js
listarPosts()
```

para atualizar a lista automaticamente.

---

## Exercício 6

Faça logout removendo:

```js
localStorage.removeItem("token")
localStorage.removeItem("usuario")
```

---

# 33. O que o aluno precisa entender

Ao terminar este exemplo, o aluno deve conseguir explicar:

1. o que é uma requisição HTTP;
2. para que serve `fetch()`;
3. a diferença entre GET, POST, PUT e DELETE;
4. o que é `response`;
5. para que serve `response.json()`;
6. para que serve `response.ok`;
7. por que usamos `async` e `await`;
8. para que serve `JSON.stringify()`;
9. o que são `headers`;
10. o que é `Content-Type`;
11. como enviar um JWT;
12. o formato `Authorization: Bearer TOKEN`;
13. como usar `try/catch`;
14. como guardar o token;
15. por que autorização precisa ser verificada no backend.

---

# 34. Regra mental para montar um `fetch`

Sempre pense nestas perguntas:

```text
1. Qual é a URL?
2. Qual é o método?
3. Preciso mandar dados?
4. Preciso mandar JSON?
5. Preciso mandar token?
6. O backend retorna JSON?
7. O que faço se response.ok for false?
```

Se souber responder essas perguntas, normalmente já consegue montar a requisição corretamente.

App.tsx
```tsx
import { useEffect, useState } from "react"

import AuthForm from "./components/AuthForm"
import PostForm from "./components/PostForm"
import PostList from "./components/PostList"

import { listarPosts } from "./services/api"


export default function App() {

    // =========================================================
    // 1. ESTADOS PRINCIPAIS DA APLICAÇÃO
    // =========================================================

    // Guarda todos os posts recebidos do backend.
    const [posts, setPosts] = useState([])


    // Guarda os dados do usuário que está logado.
    //
    // Exemplo:
    //
    // {
    //     id: 1,
    //     name: "Leonardo",
    //     email: "leo@email.com"
    // }
    //
    // Quando usuario é null, significa que ninguém está logado.
    const [usuario, setUsuario] = useState(null)


    // Guarda o JSON Web Token (JWT) recebido do backend
    // depois que o login é realizado.
    //
    // Esse token será utilizado para provar ao backend
    // que o usuário está autenticado.
    const [token, setToken] = useState(null)


    // Guarda o post que o usuário escolheu editar.
    //
    // Se for null:
    // o formulário cria um novo post.
    //
    // Se tiver um objeto:
    // o formulário edita aquele post.
    const [postEmEdicao, setPostEmEdicao] = useState(null)


    // Guarda mensagens de erro ao carregar os posts.
    const [erro, setErro] = useState("")



    // =========================================================
    // 2. RECUPERANDO LOGIN SALVO NO NAVEGADOR
    // =========================================================

    // O useEffect com [] executa apenas uma vez,
    // quando o componente App é criado.
    //
    // Aqui verificamos se o usuário já havia feito login
    // anteriormente.
    useEffect(() => {

        // 2.1 Procuramos no localStorage o token salvo.
        const tokenSalvo = localStorage.getItem("token")


        // 2.2 Procuramos também os dados do usuário.
        const usuarioSalvo = localStorage.getItem("usuario")


        // 2.3 Se os dois existem, restauramos a sessão.
        if (tokenSalvo && usuarioSalvo) {

            // Guardamos novamente o token no estado do React.
            setToken(tokenSalvo)


            // O localStorage salva somente strings.
            //
            // Por isso usamos JSON.parse() para transformar
            // a string novamente em um objeto JavaScript.
            setUsuario(JSON.parse(usuarioSalvo))
        }

    }, [])



    // =========================================================
    // 3. CARREGANDO OS POSTS QUANDO A APLICAÇÃO ABRE
    // =========================================================

    useEffect(() => {

        // Assim que o App é carregado,
        // chamamos a função responsável por buscar os posts
        // no backend.
        carregarPosts()

    }, [])



    // =========================================================
    // 4. BUSCANDO OS POSTS NO BACKEND
    // =========================================================

    async function carregarPosts() {

        try {

            setErro("")


            // 4.1 Chamamos a função listarPosts().
            //
            // Ela está no arquivo services/api.js.
            //
            // Essa função fará uma requisição:
            //
            // GET http://localhost:3000/posts
            //
            // O await faz o JavaScript esperar a resposta
            // do backend antes de continuar.
            const data = await listarPosts()


            // 4.2 Quando o backend responde,
            // colocamos os posts recebidos no estado.
            //
            // Isso faz o React renderizar novamente a tela.
            setPosts(data)

        } catch (error) {

            // 4.3 Se alguma coisa der errado na requisição,
            // mostramos a mensagem de erro.
            setErro(error.message)
        }
    }



    // =========================================================
    // 5. RECEBENDO O RESULTADO DO LOGIN
    // =========================================================

    // Essa função será chamada pelo componente AuthForm
    // depois que o backend aceitar email e senha.
    //
    // Ela recebe:
    //
    // user     -> dados do usuário
    // jwtToken -> token criado pelo backend
    function handleLogin(user, jwtToken) {

        // 5.1 Guardamos o usuário no estado.
        setUsuario(user)


        // 5.2 Guardamos o token no estado.
        //
        // Esse token poderá ser enviado para o backend
        // nas requisições protegidas.
        setToken(jwtToken)



        // =====================================================
        // 6. SALVANDO A SESSÃO NO LOCALSTORAGE
        // =====================================================

        // Se salvássemos somente no estado,
        // o login seria perdido quando a página fosse atualizada.
        //
        // Por isso também salvamos no localStorage.


        // 6.1 Salvamos o token.
        localStorage.setItem("token", jwtToken)


        // 6.2 Como usuario é um objeto,
        // precisamos convertê-lo para texto usando JSON.stringify().
        localStorage.setItem(
            "usuario",
            JSON.stringify(user)
        )
    }



    // =========================================================
    // 7. LOGOUT
    // =========================================================

    function handleLogout() {

        // 7.1 Removemos o usuário do estado.
        setUsuario(null)


        // 7.2 Removemos o token do estado.
        //
        // A partir daqui o frontend não poderá mais
        // realizar requisições protegidas.
        setToken(null)


        // 7.3 Caso estivesse editando algum post,
        // cancelamos a edição.
        setPostEmEdicao(null)


        // 7.4 Também apagamos os dados salvos no navegador.
        localStorage.removeItem("token")
        localStorage.removeItem("usuario")
    }



    // =========================================================
    // 8. DEPOIS DE CRIAR OU EDITAR UM POST
    // =========================================================

    async function handlePostSalvo() {

        // Saímos do modo de edição.
        setPostEmEdicao(null)


        // Buscamos novamente os posts no backend
        // para mostrar os dados atualizados.
        await carregarPosts()
    }



    // =========================================================
    // 9. INTERFACE
    // =========================================================

    return (
        <>

            <header className="topbar">

                <div>
                    <h1>React + Fetch</h1>
                    <p>Frontend consumindo a API de posts</p>
                </div>


                {/* 
                    9.1 Só mostramos os dados do usuário
                    e o botão Sair quando existe um usuário logado.
                */}
                {usuario && (

                    <div className="user-area">

                        <span>
                            Olá, <strong>{usuario.name}</strong>
                        </span>

                        <button
                            className="secondary"
                            onClick={handleLogout}
                        >
                            Sair
                        </button>

                    </div>

                )}

            </header>



            <main className="container">


                {/* 
                    9.2 Se NÃO existe usuário logado,
                    mostramos o formulário de login/cadastro.
                */}
                {!usuario && (

                    <AuthForm
                        onLogin={handleLogin}
                    />

                )}



                {/* 
                    9.3 Se existe usuário logado,
                    mostramos o formulário de posts.

                    Também enviamos o TOKEN para o PostForm.

                    O PostForm precisará desse token para chamar
                    rotas protegidas do backend, como:

                    POST /posts
                    PUT /posts/:id
                */}
                {usuario && (

                    <PostForm
                        token={token}
                        postEmEdicao={postEmEdicao}
                        onSalvo={handlePostSalvo}
                        onCancelar={() => setPostEmEdicao(null)}
                    />

                )}



                <div className="section-title">

                    <div>
                        <h2>Posts</h2>
                        <p>Todos os posts cadastrados no backend.</p>
                    </div>

                    <button
                        className="secondary"
                        onClick={carregarPosts}
                    >
                        Atualizar
                    </button>

                </div>



                {erro && (

                    <p className="message error">
                        {erro}
                    </p>

                )}



                {/* 
                    9.4 Enviamos para PostList:

                    posts
                        lista recebida do backend

                    usuario
                        usuário atualmente logado

                    token
                        JWT utilizado para excluir posts

                    onEditar
                        função que coloca um post em modo de edição

                    onExcluido
                        função usada para atualizar a lista
                        depois de uma exclusão
                */}
                <PostList
                    posts={posts}
                    usuario={usuario}
                    token={token}
                    onEditar={setPostEmEdicao}
                    onExcluido={carregarPosts}
                />

            </main>

        </>
    )
}
```

components/AuthForm.tsx
```tsx
import { useState } from "react"

import {
    cadastrarUsuario,
    fazerLogin
} from "../services/api"


export default function AuthForm({ onLogin }) {

    // =========================================================
    // 1. CONTROLANDO LOGIN E CADASTRO
    // =========================================================

    // O mesmo componente será utilizado para duas funções:
    //
    // login
    // cadastro
    //
    // Começamos mostrando o login.
    const [modo, setModo] = useState("login")


    // Dados preenchidos pelo usuário.
    const [name, setName] = useState("")
    const [email, setEmail] = useState("")
    const [password, setPassword] = useState("")


    // Mensagem de erro ou sucesso.
    const [mensagem, setMensagem] = useState("")


    // Controla se existe uma requisição acontecendo.
    const [carregando, setCarregando] = useState(false)



    // =========================================================
    // 2. ENVIO DO FORMULÁRIO
    // =========================================================

    async function handleSubmit(event) {

        // Impede o comportamento padrão do formulário,
        // que seria recarregar a página.
        event.preventDefault()


        setMensagem("")
        setCarregando(true)


        try {

            // =================================================
            // 3. CADASTRO
            // =================================================

            if (modo === "cadastro") {

                // 3.1 Chamamos a função cadastrarUsuario()
                // que está em services/api.js.
                //
                // Estamos enviando um objeto:
                //
                // {
                //     name,
                //     email,
                //     password
                // }
                //
                // A função transformar esse objeto em JSON
                // e enviar para:
                //
                // POST /users
                await cadastrarUsuario({
                    name,
                    email,
                    password
                })


                setMensagem(
                    "Cadastro realizado. Agora faça login."
                )


                // Voltamos para o formulário de login.
                setModo("login")


                // Limpamos alguns campos.
                setName("")
                setPassword("")


                return
            }



            // =================================================
            // 4. LOGIN
            // =================================================

            // 4.1 Enviamos email e senha para o backend.
            //
            // A função fazerLogin() fará:
            //
            // POST /auth/login
            //
            // enviando:
            //
            // {
            //     email,
            //     password
            // }
            const result = await fazerLogin({
                email,
                password
            })



            // =================================================
            // 5. RECEBENDO O TOKEN
            // =================================================

            // Se email e senha estiverem corretos,
            // esperamos que o backend responda algo parecido com:
            //
            // {
            //     user: {
            //         id: 1,
            //         name: "Leonardo",
            //         email: "leo@email.com"
            //     },
            //
            //     token: "eyJhbGciOiJIUzI1NiIs..."
            // }
            //
            // O token foi criado pelo backend.
            //
            // O frontend NÃO cria o token.
            //
            // O frontend apenas recebe, guarda e envia
            // esse token posteriormente.


            // 5.1 Chamamos a função onLogin recebida do App.
            //
            // Estamos mandando:
            //
            // result.user  -> dados do usuário
            // result.token -> JSON Web Token
            //
            // Quem vai salvar esses dados será o App.jsx.
            onLogin(
                result.user,
                result.token
            )


        } catch (error) {

            // Se o backend responder com erro,
            // mostramos a mensagem.
            setMensagem(error.message)

        } finally {

            // Executa tanto em caso de sucesso quanto erro.
            setCarregando(false)
        }
    }



    return (

        <section className="card auth-card">

            <h2>
                {modo === "login"
                    ? "Entrar"
                    : "Criar conta"}
            </h2>


            <form onSubmit={handleSubmit}>


                {modo === "cadastro" && (

                    <label>

                        Nome

                        <input
                            type="text"
                            value={name}
                            onChange={(event) =>
                                setName(event.target.value)
                            }
                            required
                        />

                    </label>

                )}



                <label>

                    E-mail

                    <input
                        type="email"
                        value={email}
                        onChange={(event) =>
                            setEmail(event.target.value)
                        }
                        required
                    />

                </label>



                <label>

                    Senha

                    <input
                        type="password"
                        value={password}
                        onChange={(event) =>
                            setPassword(event.target.value)
                        }
                        required
                    />

                </label>



                <button
                    type="submit"
                    disabled={carregando}
                >

                    {
                        carregando
                            ? "Aguarde..."
                            : modo === "login"
                                ? "Entrar"
                                : "Cadastrar"
                    }

                </button>

            </form>



            {mensagem && (

                <p className="message">
                    {mensagem}
                </p>

            )}



            <button
                className="link-button"
                onClick={() => {

                    setModo(
                        modo === "login"
                            ? "cadastro"
                            : "login"
                    )

                    setMensagem("")
                }}
            >

                {
                    modo === "login"
                        ? "Ainda não tenho conta"
                        : "Já tenho uma conta"
                }

            </button>

        </section>
    )
}

```

components/PostForm.jsx
```jsx
import { useEffect, useState } from "react"

import {
    atualizarPost,
    criarPost
} from "../services/api"


export default function PostForm({
    token,
    postEmEdicao,
    onSalvo,
    onCancelar
}) {

    // =========================================================
    // 1. ESTADOS DO FORMULÁRIO
    // =========================================================

    const [title, setTitle] = useState("")
    const [content, setContent] = useState("")

    const [mensagem, setMensagem] = useState("")
    const [carregando, setCarregando] = useState(false)



    // =========================================================
    // 2. VERIFICANDO SE ESTAMOS CRIANDO OU EDITANDO
    // =========================================================

    useEffect(() => {

        // Se recebemos um postEmEdicao,
        // significa que o usuário clicou no botão Editar.
        if (postEmEdicao) {

            // Preenchemos o formulário com os dados atuais
            // daquele post.
            setTitle(postEmEdicao.title)
            setContent(postEmEdicao.content)

        } else {

            // Se não existe postEmEdicao,
            // deixamos o formulário vazio.
            setTitle("")
            setContent("")
        }

    }, [postEmEdicao])



    // =========================================================
    // 3. ENVIANDO O FORMULÁRIO
    // =========================================================

    async function handleSubmit(event) {

        event.preventDefault()

        setMensagem("")
        setCarregando(true)


        try {

            // =================================================
            // 4. ATUALIZANDO UM POST
            // =================================================

            if (postEmEdicao) {

                // Chamamos atualizarPost() enviando:
                //
                // 1º parâmetro:
                // ID do post
                //
                // 2º parâmetro:
                // novos dados
                //
                // 3º parâmetro:
                // TOKEN do usuário
                //
                // O token é obrigatório porque editar um post
                // é uma ação protegida pelo backend.
                await atualizarPost(

                    postEmEdicao.id,

                    {
                        title,
                        content
                    },

                    token
                )


            } else {

                // =================================================
                // 5. CRIANDO UM POST
                // =================================================

                // Para criar também precisamos do token.
                //
                // O backend precisa descobrir qual usuário
                // está realizando a operação.
                //
                // Por isso enviamos:
                //
                // dados do post
                // +
                // token do usuário
                await criarPost(

                    {
                        title,
                        content
                    },

                    token
                )
            }



            // =================================================
            // 6. DEPOIS QUE O BACKEND RESPONDE COM SUCESSO
            // =================================================

            // Limpamos o formulário.
            setTitle("")
            setContent("")


            // Avisamos o componente App que o post foi salvo.
            //
            // O App então busca novamente a lista de posts
            // no backend.
            onSalvo()


        } catch (error) {

            setMensagem(error.message)

        } finally {

            setCarregando(false)
        }
    }



    return (

        <section className="card">

            <h2>
                {
                    postEmEdicao
                        ? "Editar post"
                        : "Novo post"
                }
            </h2>



            <form onSubmit={handleSubmit}>


                <label>

                    Título

                    <input
                        type="text"
                        value={title}
                        onChange={(event) =>
                            setTitle(event.target.value)
                        }
                        required
                    />

                </label>



                <label>

                    Conteúdo

                    <textarea
                        value={content}
                        onChange={(event) =>
                            setContent(event.target.value)
                        }
                        required
                        rows="6"
                    />

                </label>



                <div className="actions">

                    <button
                        type="submit"
                        disabled={carregando}
                    >

                        {
                            carregando
                                ? "Salvando..."
                                : postEmEdicao
                                    ? "Salvar alterações"
                                    : "Publicar"
                        }

                    </button>



                    {postEmEdicao && (

                        <button
                            type="button"
                            className="secondary"
                            onClick={onCancelar}
                        >
                            Cancelar
                        </button>

                    )}

                </div>

            </form>



            {mensagem && (

                <p className="message">
                    {mensagem}
                </p>

            )}

        </section>
    )
}
```

components/PostList.jsx
```jsx
import { Pencil, Trash2 } from "lucide-react"

import { excluirPost } from "../services/api"


export default function PostList({
    posts,
    usuario,
    token,
    onEditar,
    onExcluido
}) {


    // =========================================================
    // 1. EXCLUINDO UM POST
    // =========================================================

    async function handleExcluir(post) {

        // Antes de excluir, pedimos confirmação.
        const confirmou = window.confirm(
            `Deseja realmente excluir "${post.title}"?`
        )


        if (!confirmou) {
            return
        }


        try {

            // =================================================
            // 2. CHAMANDO O BACKEND
            // =================================================

            // Para excluir precisamos enviar:
            //
            // post.id
            //     informa QUAL post deve ser excluído
            //
            // token
            //     informa QUEM está fazendo a requisição
            //
            // A função excluirPost() enviará:
            //
            // DELETE /posts/:id
            //
            // junto com o token.
            await excluirPost(
                post.id,
                token
            )


            // =================================================
            // 3. ATUALIZANDO A LISTA
            // =================================================

            // Depois da exclusão,
            // avisamos o App para buscar novamente os posts.
            onExcluido()


        } catch (error) {

            alert(error.message)
        }
    }



    if (posts.length === 0) {

        return (

            <section className="card">
                <p>Nenhum post cadastrado ainda.</p>
            </section>

        )
    }



    return (

        <section className="posts">

            {posts.map(post => {


                // =================================================
                // 4. VERIFICANDO SE O POST PERTENCE AO USUÁRIO
                // =================================================

                // Para mostrar os botões Editar e Excluir,
                // comparamos:
                //
                // ID do usuário dono do post
                //
                // com
                //
                // ID do usuário atualmente logado.
                //
                // Exemplo:
                //
                // post.user.id = 3
                // usuario.id   = 3
                //
                // então ehDono será true.
                const ehDono =
                    usuario &&
                    post.user &&
                    post.user.id === usuario.id



                return (

                    <article
                        className="card post"
                        key={post.id}
                    >

                        <div className="post-header">

                            <div>

                                <h2>
                                    {post.title}
                                </h2>

                                <small>
                                    por {post.user?.name || "Usuário"}
                                </small>

                            </div>



                            {/* 
                                5. Só mostramos os botões
                                se o usuário logado for o dono do post.

                                IMPORTANTE:

                                Essa verificação no frontend melhora
                                a interface do usuário.

                                Porém ela NÃO substitui a segurança
                                do backend.

                                O backend também precisa verificar
                                o token e conferir se aquele usuário
                                realmente é o dono do post.
                            */}
                            {ehDono && (

                                <div className="post-actions">


                                    <button
                                        className="icon-button"
                                        onClick={() =>
                                            onEditar(post)
                                        }
                                        title="Editar"
                                    >
                                        <Pencil size={18} />
                                    </button>



                                    <button
                                        className="icon-button danger"
                                        onClick={() =>
                                            handleExcluir(post)
                                        }
                                        title="Excluir"
                                    >
                                        <Trash2 size={18} />
                                    </button>

                                </div>

                            )}

                        </div>



                        <p className="post-content">
                            {post.content}
                        </p>

                    </article>

                )
            })}

        </section>
    )
}

```

services/api.js
```js
// =============================================================
// 1. ENDEREÇO DO BACKEND
// =============================================================

// Aqui colocamos o endereço principal da nossa API.
//
// Nosso frontend React está rodando em um endereço.
//
// Exemplo:
//
// http://localhost:5173
//
// Enquanto nosso backend está rodando em outro:
//
// http://localhost:3000
//
// Quando usamos fetch(), o frontend envia uma requisição
// HTTP para esse backend.
const API_URL = "http://localhost:3000"



// =============================================================
// 2. CADASTRO DE USUÁRIO
// =============================================================

export async function cadastrarUsuario(data) {

    // =========================================================
    // 2.1 O QUE TEM DENTRO DE "data"?
    // =========================================================

    // Essa função recebe um objeto vindo do AuthForm.
    //
    // Exemplo:
    //
    // {
    //     name: "Leonardo",
    //     email: "leo@email.com",
    //     password: "123456"
    // }



    // =========================================================
    // 2.2 FAZENDO A REQUISIÇÃO PARA O BACKEND
    // =========================================================

    // fetch() é uma função do JavaScript utilizada
    // para fazer requisições HTTP.
    //
    // Ela recebe principalmente dois parâmetros:
    //
    // 1º parâmetro:
    // endereço da requisição
    //
    // 2º parâmetro:
    // configurações da requisição
    const response = await fetch(
        `${API_URL}/users`,
        {

            // -------------------------------------------------
            // 2.3 MÉTODO HTTP
            // -------------------------------------------------

            // POST é utilizado normalmente quando queremos
            // criar um novo recurso.
            //
            // Neste caso:
            // criar um novo usuário.
            method: "POST",



            // -------------------------------------------------
            // 2.4 HEADERS
            // -------------------------------------------------

            // Headers são informações adicionais enviadas
            // junto com a requisição.
            headers: {

                // Estamos avisando ao backend que o corpo
                // da requisição está em formato JSON.
                "Content-Type": "application/json"
            },



            // -------------------------------------------------
            // 2.5 BODY
            // -------------------------------------------------

            // O body é o corpo da requisição.
            //
            // É onde enviamos os dados.
            //
            // Porém o fetch não envia diretamente
            // um objeto JavaScript.
            //
            // Por isso usamos JSON.stringify().
            //
            // Ele transforma:
            //
            // {
            //     name: "Leonardo"
            // }
            //
            // em:
            //
            // '{"name":"Leonardo"}'
            body: JSON.stringify(data)
        }
    )



    // =========================================================
    // 2.6 RECEBENDO A RESPOSTA
    // =========================================================

    // response contém várias informações da resposta HTTP.
    //
    // Entre elas:
    //
    // response.status
    // response.ok
    //
    // Porém o conteúdo JSON enviado pelo backend
    // precisa ser convertido.
    //
    // response.json() transforma o JSON recebido
    // em um objeto JavaScript.
    const result = await response.json()



    // =========================================================
    // 2.7 VERIFICANDO ERROS
    // =========================================================

    // response.ok será true para respostas de sucesso.
    //
    // Normalmente status entre 200 e 299.
    //
    // Exemplos:
    //
    // 200 OK
    // 201 Created
    //
    // Será false em erros como:
    //
    // 400 Bad Request
    // 401 Unauthorized
    // 404 Not Found
    // 500 Internal Server Error
    if (!response.ok) {

        throw new Error(
            result.message ||
            "Erro ao cadastrar usuário."
        )
    }



    // =========================================================
    // 2.8 DEVOLVENDO A RESPOSTA
    // =========================================================

    return result
}



// =============================================================
// 3. LOGIN
// =============================================================

export async function fazerLogin(data) {

    // =========================================================
    // 3.1 ENVIANDO EMAIL E SENHA
    // =========================================================

    // Recebemos algo parecido com:
    //
    // {
    //     email: "leo@email.com",
    //     password: "123456"
    // }
    //
    // E enviamos para:
    //
    // POST /auth/login

    const response = await fetch(
        `${API_URL}/auth/login`,
        {

            method: "POST",

            headers: {
                "Content-Type": "application/json"
            },

            body: JSON.stringify(data)
        }
    )



    // =========================================================
    // 3.2 RECEBENDO A RESPOSTA DO BACKEND
    // =========================================================

    const result = await response.json()



    if (!response.ok) {

        throw new Error(
            result.message ||
            "Erro ao fazer login."
        )
    }



    // =========================================================
    // 3.3 O BACKEND CRIA O TOKEN
    // =========================================================

    // Se email e senha estiverem corretos,
    // o backend normalmente cria um JSON Web Token (JWT).
    //
    // A resposta pode ser parecida com:
    //
    // {
    //     user: {
    //         id: 1,
    //         name: "Leonardo",
    //         email: "leo@email.com"
    //     },
    //
    //     token: "eyJhbGciOiJIUzI1NiIs..."
    // }
    //
    // IMPORTANTE:
    //
    // O TOKEN É CRIADO PELO BACKEND.
    //
    // O frontend apenas:
    //
    // 1. recebe
    // 2. guarda
    // 3. envia novamente quando necessário.

    return result
}



// =============================================================
// 4. LISTAR POSTS
// =============================================================

export async function listarPosts() {

    // =========================================================
    // 4.1 ROTA PÚBLICA
    // =========================================================

    // Para listar os posts não precisamos enviar token.
    //
    // Isso significa que estamos considerando:
    //
    // GET /posts
    //
    // uma rota pública.
    //
    // Até alguém que não está logado pode visualizar os posts.
    const response = await fetch(
        `${API_URL}/posts`
    )



    const result = await response.json()



    if (!response.ok) {

        throw new Error(
            result.message ||
            "Erro ao carregar posts."
        )
    }



    return result
}



// =============================================================
// 5. CRIAR POST
// =============================================================

export async function criarPost(data, token) {

    // =========================================================
    // 5.1 ESTA É UMA ROTA PROTEGIDA
    // =========================================================

    // Para criar um post precisamos estar autenticados.
    //
    // O backend precisa saber:
    //
    // "Quem está tentando criar este post?"
    //
    // Para isso enviamos o token recebido durante o login.



    const response = await fetch(
        `${API_URL}/posts`,
        {

            method: "POST",



            headers: {

                // -------------------------------------------------
                // 5.2 TIPO DO CONTEÚDO
                // -------------------------------------------------

                "Content-Type": "application/json",



                // -------------------------------------------------
                // 5.3 ENVIANDO O TOKEN
                // -------------------------------------------------

                // O token é enviado normalmente no header:
                //
                // Authorization
                //
                // seguindo este formato:
                //
                // Authorization: Bearer TOKEN
                //
                // Exemplo:
                //
                // Authorization:
                // Bearer eyJhbGciOiJIUzI1NiIs...
                //
                // "Bearer" significa que estamos apresentando
                // um token de acesso.
                "Authorization": `Bearer ${token}`
            },



            // -------------------------------------------------
            // 5.4 DADOS DO POST
            // -------------------------------------------------

            // Exemplo:
            //
            // {
            //     title: "Meu post",
            //     content: "Conteúdo do post"
            // }
            body: JSON.stringify(data)
        }
    )



    const result = await response.json()



    if (!response.ok) {

        throw new Error(
            result.message ||
            "Erro ao criar post."
        )
    }



    return result
}



// =============================================================
// 6. O QUE ACONTECE COM O TOKEN NO BACKEND?
// =============================================================

// Quando essa requisição chegar ao backend:
//
// POST /posts
//
// teremos um header parecido com:
//
// Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
//
// O middleware de autenticação do backend deverá:
//
// 1. pegar o header Authorization
//
// 2. separar a palavra "Bearer" do token
//
// 3. pegar apenas o token
//
// 4. verificar se o token é válido
//
// 5. descobrir qual usuário está dentro do token
//
// 6. permitir ou bloquear a requisição
//
// Portanto:
//
// FRONTEND
//     envia o token
//
// BACKEND
//     valida o token



// =============================================================
// 7. ATUALIZAR POST
// =============================================================

export async function atualizarPost(
    id,
    data,
    token
) {

    // =========================================================
    // 7.1 MONTANDO O ENDEREÇO
    // =========================================================

    // Se:
    //
    // id = 5
    //
    // a URL será:
    //
    // http://localhost:3000/posts/5


    const response = await fetch(
        `${API_URL}/posts/${id}`,
        {

            method: "PUT",



            headers: {

                // Como estamos enviando title e content,
                // informamos que o conteúdo será JSON.
                "Content-Type": "application/json",


                // Como editar é uma operação protegida,
                // enviamos novamente o JWT.
                "Authorization": `Bearer ${token}`
            },



            // Novos dados do post.
            body: JSON.stringify(data)
        }
    )



    const result = await response.json()



    if (!response.ok) {

        throw new Error(
            result.message ||
            "Erro ao atualizar post."
        )
    }



    return result
}



// =============================================================
// 8. SEGURANÇA AO EDITAR
// =============================================================

// O frontend pode esconder o botão "Editar"
// de quem não é dono do post.
//
// Porém isso NÃO é segurança suficiente.
//
// Uma pessoa poderia abrir ferramentas como:
//
// Postman
// Insomnia
// Thunder Client
//
// e tentar manualmente:
//
// PUT /posts/5
//
// Por isso o backend precisa:
//
// 1. validar o token
//
// 2. descobrir o ID do usuário autenticado
//
// 3. buscar o post
//
// 4. verificar quem é o dono do post
//
// 5. comparar:
//
// post.user.id
//
// com:
//
// usuário autenticado
//
// 6. somente depois permitir a edição.



// =============================================================
// 9. EXCLUIR POST
// =============================================================

export async function excluirPost(
    id,
    token
) {

    const response = await fetch(
        `${API_URL}/posts/${id}`,
        {

            // DELETE informa ao backend
            // que queremos remover um recurso.
            method: "DELETE",



            headers: {

                // Mesmo não existindo body,
                // precisamos enviar o token,
                // porque excluir é uma operação protegida.
                "Authorization": `Bearer ${token}`
            }
        }
    )



    // =========================================================
    // 9.1 STATUS 204
    // =========================================================

    // Uma exclusão normalmente pode retornar:
    //
    // 204 No Content
    //
    // Isso significa:
    //
    // "A operação funcionou, mas não existe
    // conteúdo no corpo da resposta."
    //
    // Por isso NÃO fazemos diretamente:
    //
    // const result = await response.json()
    //
    // Se tentarmos transformar uma resposta vazia
    // em JSON, pode ocorrer erro.



    // =========================================================
    // 9.2 SE A EXCLUSÃO DER ERRADO
    // =========================================================

    if (!response.ok) {

        // Nesse caso esperamos que o backend
        // tenha enviado uma mensagem JSON.
        const result = await response.json()


        throw new Error(
            result.message ||
            "Erro ao excluir post."
        )
    }
}



// =============================================================
// 10. RESUMO DO FLUXO DE AUTENTICAÇÃO
// =============================================================

// LOGIN:
//
// 1. O usuário digita email e senha.
//
// 2. O React chama:
//
//    fazerLogin()
//
// 3. fazerLogin() envia:
//
//    POST /auth/login
//
// 4. O backend verifica email e senha.
//
// 5. Se estiverem corretos,
//    o backend cria um JSON Web Token.
//
// 6. O backend responde:
//
//    {
//        user: {...},
//        token: "..."
//    }
//
// 7. O React recebe o token.
//
// 8. O App salva o token no estado.
//
// 9. Também salvamos o token no localStorage.
//
//
//
// REQUISIÇÃO PROTEGIDA:
//
// 10. O usuário tenta criar, editar ou excluir.
//
// 11. O frontend recupera o token.
//
// 12. O fetch envia:
//
//     Authorization: Bearer TOKEN
//
// 13. A requisição chega ao backend.
//
// 14. O middleware de autenticação
//     pega o header Authorization.
//
// 15. O middleware verifica o JWT.
//
// 16. Se o token for válido,
//     o backend descobre qual usuário está autenticado.
//
// 17. A requisição continua.
//
// 18. O controller/service executa a operação.
//
//
//
// TOKEN INVÁLIDO:
//
// Se o token:
//
// - não existir
// - estiver errado
// - estiver expirado
//
// o backend normalmente responde:
//
// 401 Unauthorized
//
// e a operação é bloqueada.

```
