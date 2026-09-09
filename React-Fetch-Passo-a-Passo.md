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

    const [posts, setPosts] = useState([])
    const [usuario, setUsuario] = useState(null)
    const [token, setToken] = useState(null)

    const [postEmEdicao, setPostEmEdicao] = useState(null)
    const [erro, setErro] = useState("")


    // Quando a aplicação abre, recuperamos os dados salvos
    // no navegador.
    useEffect(() => {

        const tokenSalvo = localStorage.getItem("token")
        const usuarioSalvo = localStorage.getItem("usuario")

        if (tokenSalvo && usuarioSalvo) {

            setToken(tokenSalvo)
            setUsuario(JSON.parse(usuarioSalvo))
        }

    }, [])


    // Carrega os posts quando a aplicação inicia.
    useEffect(() => {

        carregarPosts()

    }, [])


    async function carregarPosts() {

        try {

            setErro("")

            const data = await listarPosts()

            setPosts(data)

        } catch (error) {

            setErro(error.message)
        }
    }


    function handleLogin(user, jwtToken) {

        setUsuario(user)
        setToken(jwtToken)

        // Guardamos o token e o usuário no navegador.
        localStorage.setItem("token", jwtToken)
        localStorage.setItem("usuario", JSON.stringify(user))
    }


    function handleLogout() {

        setUsuario(null)
        setToken(null)
        setPostEmEdicao(null)

        localStorage.removeItem("token")
        localStorage.removeItem("usuario")
    }


    async function handlePostSalvo() {

        setPostEmEdicao(null)

        await carregarPosts()
    }


    return (
        <>

            <header className="topbar">

                <div>
                    <h1>React + Fetch</h1>
                    <p>Frontend consumindo a API de posts</p>
                </div>

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

                {!usuario && (
                    <AuthForm onLogin={handleLogin} />
                )}


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

    const [modo, setModo] = useState("login")

    const [name, setName] = useState("")
    const [email, setEmail] = useState("")
    const [password, setPassword] = useState("")

    const [mensagem, setMensagem] = useState("")
    const [carregando, setCarregando] = useState(false)


    async function handleSubmit(event) {

        event.preventDefault()

        setMensagem("")
        setCarregando(true)

        try {

            if (modo === "cadastro") {

                await cadastrarUsuario({
                    name,
                    email,
                    password
                })

                setMensagem("Cadastro realizado. Agora faça login.")
                setModo("login")
                setName("")
                setPassword("")

                return
            }

            const result = await fazerLogin({
                email,
                password
            })

            onLogin(result.user, result.token)

        } catch (error) {

            setMensagem(error.message)

        } finally {

            setCarregando(false)
        }
    }


    return (
        <section className="card auth-card">

            <h2>
                {modo === "login" ? "Entrar" : "Criar conta"}
            </h2>

            <form onSubmit={handleSubmit}>

                {modo === "cadastro" && (
                    <label>
                        Nome

                        <input
                            type="text"
                            value={name}
                            onChange={(event) => setName(event.target.value)}
                            required
                        />
                    </label>
                )}

                <label>
                    E-mail

                    <input
                        type="email"
                        value={email}
                        onChange={(event) => setEmail(event.target.value)}
                        required
                    />
                </label>

                <label>
                    Senha

                    <input
                        type="password"
                        value={password}
                        onChange={(event) => setPassword(event.target.value)}
                        required
                    />
                </label>

                <button type="submit" disabled={carregando}>
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
                    setModo(modo === "login" ? "cadastro" : "login")
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

    const [title, setTitle] = useState("")
    const [content, setContent] = useState("")
    const [mensagem, setMensagem] = useState("")
    const [carregando, setCarregando] = useState(false)


    useEffect(() => {

        if (postEmEdicao) {

            setTitle(postEmEdicao.title)
            setContent(postEmEdicao.content)

        } else {

            setTitle("")
            setContent("")
        }

    }, [postEmEdicao])


    async function handleSubmit(event) {

        event.preventDefault()

        setMensagem("")
        setCarregando(true)

        try {

            if (postEmEdicao) {

                await atualizarPost(
                    postEmEdicao.id,
                    {
                        title,
                        content
                    },
                    token
                )

            } else {

                await criarPost(
                    {
                        title,
                        content
                    },
                    token
                )
            }

            setTitle("")
            setContent("")

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
                {postEmEdicao ? "Editar post" : "Novo post"}
            </h2>

            <form onSubmit={handleSubmit}>

                <label>
                    Título

                    <input
                        type="text"
                        value={title}
                        onChange={(event) => setTitle(event.target.value)}
                        required
                    />
                </label>

                <label>
                    Conteúdo

                    <textarea
                        value={content}
                        onChange={(event) => setContent(event.target.value)}
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

    async function handleExcluir(post) {

        const confirmou = window.confirm(
            `Deseja realmente excluir "${post.title}"?`
        )

        if (!confirmou) {
            return
        }

        try {

            await excluirPost(post.id, token)

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

                const ehDono =
                    usuario &&
                    post.user &&
                    post.user.id === usuario.id

                return (
                    <article className="card post" key={post.id}>

                        <div className="post-header">

                            <div>
                                <h2>{post.title}</h2>

                                <small>
                                    por {post.user?.name || "Usuário"}
                                </small>
                            </div>

                            {ehDono && (
                                <div className="post-actions">

                                    <button
                                        className="icon-button"
                                        onClick={() => onEditar(post)}
                                        title="Editar"
                                    >
                                        <Pencil size={18} />
                                    </button>

                                    <button
                                        className="icon-button danger"
                                        onClick={() => handleExcluir(post)}
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
const API_URL = "http://localhost:3000"


// =========================================================
// CADASTRO
// =========================================================

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
        throw new Error(result.message || "Erro ao cadastrar usuário.")
    }

    return result
}


// =========================================================
// LOGIN
// =========================================================

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


// =========================================================
// LISTAR POSTS
// =========================================================

export async function listarPosts() {

    const response = await fetch(`${API_URL}/posts`)

    const result = await response.json()

    if (!response.ok) {
        throw new Error(result.message || "Erro ao carregar posts.")
    }

    return result
}


// =========================================================
// CRIAR POST
// =========================================================

export async function criarPost(data, token) {

    const response = await fetch(`${API_URL}/posts`, {
        method: "POST",

        headers: {
            "Content-Type": "application/json",

            // O backend espera:
            // Authorization: Bearer TOKEN
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


// =========================================================
// ATUALIZAR POST
// =========================================================

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
        throw new Error(result.message || "Erro ao atualizar post.")
    }

    return result
}


// =========================================================
// EXCLUIR POST
// =========================================================

export async function excluirPost(id, token) {

    const response = await fetch(`${API_URL}/posts/${id}`, {
        method: "DELETE",

        headers: {
            "Authorization": `Bearer ${token}`
        }
    })

    // DELETE retorna 204 No Content.
    // Nesse caso não existe JSON no corpo da resposta.
    if (!response.ok) {

        const result = await response.json()

        throw new Error(result.message || "Erro ao excluir post.")
    }
}

```
