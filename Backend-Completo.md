# JWT Backend (sem camada Repository) — Código atualizado

Backend em **Node.js + TypeScript + Express + TypeORM + MySQL**, com autenticação por **JSON Web Token (JWT)**.

Arquitetura utilizada:

```text
Model → Service → Controller → Routes
```

A parte de **Post** está completa e integrada ao JWT. O usuário autenticado é identificado pelo token, portanto o cliente não escolhe manualmente o `userId` ao criar, editar ou excluir um post.

---

## 📦 Instalação

```bash
npm init -y
npm install express cors dotenv typeorm mysql2 reflect-metadata bcrypt jsonwebtoken
npm install --save-dev typescript ts-node-dev @types/node @types/express @types/cors @types/bcrypt @types/jsonwebtoken
```

Crie o banco:

```sql
CREATE DATABASE jwt_backend;
```

Execute:

```bash
npm run dev
```

---

## 📁 Estrutura de pastas

```text
src/
  config/
    data-source.ts

  controllers/
    AuthController.ts
    PostController.ts
    UserController.ts

  middlewares/
    authMiddleware.ts
    errorHandler.ts
    validatePost.ts
    validateUser.ts

  models/
    Post.ts
    User.ts

  routes/
    auth.routes.ts
    index.ts
    post.routes.ts
    user.routes.ts

  services/
    PostService.ts
    UserService.ts

  utils/
    jwt.ts
    omitPassword.ts

  server.ts
```

---

## 🔐 Como o JWT é usado

No login, o backend gera um token contendo o `id` e o `email` do usuário. Nas rotas protegidas, `authMiddleware` valida esse token e salva os dados decodificados em `req.user`.

Assim, nos Controllers podemos obter o usuário autenticado desta forma:

```ts
const userId = (req as any).user.id
```

Para posts:

- `POST /posts`: cria o post para o usuário do JWT.
- `PUT /posts/:id`: o ID da URL é o ID do **post**; o usuário vem do JWT.
- `DELETE /posts/:id`: o ID da URL é o ID do **post**; o usuário vem do JWT.
- O Service verifica se o post realmente pertence ao usuário autenticado antes de permitir alteração ou exclusão.

---

## 📄 Arquivos

### Instalação e configuração
**`package.json`**

```json
{
  "name": "backend",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "ts-node-dev src/server.ts"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "type": "commonjs",
  "dependencies": {
    "@types/dotenv": "^6.1.1",
    "bcrypt": "^6.0.0",
    "cors": "^2.8.6",
    "dotenv": "^17.4.2",
    "express": "^5.2.1",
    "jsonwebtoken": "^9.0.3",
    "mysql2": "^3.24.2",
    "reflect-metadata": "^0.2.2",
    "typeorm": "^1.1.0"
  },
  "devDependencies": {
    "@types/bcrypt": "^6.0.0",
    "@types/cors": "^2.8.19",
    "@types/express": "^5.0.6",
    "@types/jsonwebtoken": "^9.0.10",
    "@types/node": "^26.4.0",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.9.2"
  }
}
```
**`tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```
**`.env`**

```text
# Servidor
PORT=3000

# Banco de dados (ajuste com os dados do seu MySQL)
DB_HOST=localhost
DB_PORT=3306
DB_USERNAME=root
DB_PASSWORD=root
DB_DATABASE=jwt_backend

# JWT
JWT_SECRET=minhaChaveSecreta123
JWT_EXPIRES_IN=86400
```
> O `.env` deve permanecer no `.gitignore`. Em projetos reais, não publique `JWT_SECRET`, senhas ou credenciais do banco.

---

### Conexão com o banco

**`src\config\data-source.ts`**

```ts
import "reflect-metadata"
import { DataSource } from "typeorm"
import * as dotenv from 'dotenv'
import { User } from "../models/User"
import { Post } from "../models/Post"

// sempre use isso quando for trabalhar com dotenv
// ele carrega as informações do .env para o objeto process.env
dotenv.config()

export const AppDataSource = new DataSource({
    type: "mysql",
    host: process.env.DB_HOST,
    port: Number(process.env.DB_PORT),
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_DATABASE,
    // Em desenvolvimento facilita, pois cria/atualiza as tabelas automaticamente
    // Em produção, o ideal é usar migrations em vez de synchronize
    synchronize: true,
    logging: false,
    entities: [User, Post]
})
```
---

## Models

### User

**`src\models\User.ts`**

```ts
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from "typeorm"
import { Post } from "./Post"

@Entity("users")
export class User {
    @PrimaryGeneratedColumn()
    id!: number

    @Column()
    name!: string

    @Column({ unique: true })
    email!: string

    @Column()
    password!: string

    // Um usuário pode ter vários posts
    @OneToMany(() => Post, (post) => post.user)
    posts!: Post[]
}
```
### Post

O `ManyToOne` representa que vários posts podem pertencer ao mesmo usuário. O `onDelete: "CASCADE"` remove os posts associados caso o usuário seja excluído.

**`src\models\Post.ts`**

```ts
import {
    Entity,
    PrimaryGeneratedColumn,
    Column,
    ManyToOne
} from "typeorm"

import { User } from "./User"

@Entity("posts")
export class Post {

    @PrimaryGeneratedColumn()
    id!: number

    @Column()
    title!: string

    @Column({ type: "text" })
    content!: string

    // Cada post pertence a um usuário
    @ManyToOne(
        () => User,
        (user) => user.posts,
        {
            onDelete: "CASCADE"
        }
    )
    user!: User
}
```
---

## Utils

### JWT

**`src\utils\jwt.ts`**

```ts
import jwt from "jsonwebtoken"
import * as dotenv from 'dotenv'

dotenv.config()

interface Payload {
    id: number
    email: string
}

// gera um token
export function generateToken(payload: Payload) {
    return jwt.sign(payload, process.env.JWT_SECRET!, {
        expiresIn: Number(process.env.JWT_EXPIRES_IN)
    })
}

// verifica se o token é valido
export function verifyToken(token: string) {
    try {
        return jwt.verify(token, process.env.JWT_SECRET!)
    } catch {
        return null
    }
}
```
### Remover senha do retorno

**`src\utils\omitPassword.ts`**

```ts
import { User } from "../models/User"

// Remove a senha do objeto de usuário antes de retornar para o cliente
export function omitPassword(user: User) {
    const { password, ...userWithoutPassword } = user
    return userWithoutPassword
}
```
---

## Services

A camada Service contém as regras de negócio e acessa o TypeORM diretamente.

### UserService

**`src\services\UserService.ts`**

```ts
import { AppDataSource } from "../config/data-source"
import { User } from "../models/User"
import bcrypt from "bcrypt"
import { omitPassword } from "../utils/omitPassword"
import { generateToken } from "../utils/jwt"

// A camada Service é responsável por chamar os métodos de Repository e cuidar das validações das nossas regras de negócio
// Como não estamos usando uma camada Repository separada, o Service acessa o TypeORM diretamente aqui dentro

// repo é um objeto do TypeORM que contém todas as funções que precisamos para trabalhar com o banco, ligado a uma entidade específica (nesse caso, User)
const repo = AppDataSource.getRepository(User)

// Aqui estamos criando uma classe de erro que extende a classe Error
// Isso é para permitir que o errorHandler identifique o tipo de erro de uma forma mais clara
export class NotFoundError extends Error {}
export class UnauthorizedError extends Error {}

export const UserService = {

    // O Controller NÃO PODE se comunicar diretamente com o banco, e sim com o Service
    async listAll() {
        // o método find() vem do TypeORM. Ele procura algo em uma tabela
        // ele aceita como parâmetro um objeto com opções para esta busca
        // nesse nosso caso, estamos buscando também os posts relacionados com este usuário
        const users = await repo.find({
            relations: {
                posts: true
            }
        })

        return users.map(user => omitPassword(user))
    },

    async getById(id: number) {
        const user = await repo.findOne({
            where: { id },
            relations: {
                posts: true
            }
        })

        // Se não encontrarmos um user com esse id, ele não existe
        if (!user) {
            throw new NotFoundError("Usuário não encontrado!")
        }

        return omitPassword(user)
    },

    async create(data: { name: string, email: string, password: string }) {
        // Este método gera uma senha criptografada
        const hashedPassword = await bcrypt.hash(data.password, 10)

        // cria o usuário
        const user = repo.create({
            name: data.name,
            email: data.email,
            password: hashedPassword
        })

        // salva ele no banco
        const savedUser = await repo.save(user)

        // Retornamos o usuário sem a senha
        return omitPassword(savedUser)
    },

    async login(data: { email: string, password: string }) {
        // Primeiro buscamos o usuário pelo email
        // Esse findOne por email será usado no login
        const user = await repo.findOne({
            where: {
                email: data.email
            }
        })

        // Se não encontrou usuário com esse email, lançamos erro
        if (!user) {
            throw new NotFoundError("Usuário não encontrado!")
        }

        // Agora comparamos a senha enviada com a senha criptografada no banco
        const passwordIsValid = await bcrypt.compare(data.password, user.password)

        // Se a senha estiver errada, lançamos erro de autorização
        if (!passwordIsValid) {
            throw new UnauthorizedError("Senha inválida!")
        }

        // Se chegou até aqui, email e senha estão corretos
        // Então podemos gerar o token JWT
        const token = generateToken({
            id: user.id,
            email: user.email
        })

        // Retornamos o usuário sem senha e o token
        return {
            user: omitPassword(user),
            token
        }
    },


    // =========================================================
    // MÉTODO ANTIGO
    //
    // Ele recebia qualquer id enviado pelo Controller.
    //
    // Como o Controller antigo pegava esse id da URL,
    // um usuário poderia tentar alterar outro usuário.
    // =========================================================

    /*
    async update(id: number, data: { name?: string, email?: string, password?: string }) {
        // encontra o usuário pelo id
        const user = await repo.findOne({
            where: {
                id
            }
        })

        if (!user) {
            throw new NotFoundError("Usuário não encontrado!")
        }

        // Só vamos alterar/atualizar os campos que vierem
        if (data.name) user.name = data.name
        if (data.email) user.email = data.email

        // Se vier uma senha nova, a gente precisa criptografar ela de novo
        if (data.password) {
            user.password = await bcrypt.hash(data.password, 10)
        }

        // Depois de tudo isso acima, salvamos de novo
        // Como o user já possui id, o TypeORM entende que é atualização, não novo cadastro
        const updatedUser = await repo.save(user)

        // Retorna o usuário sem a senha
        return omitPassword(updatedUser)
    },
    */


    // =========================================================
    // NOVO MÉTODO
    //
    // O id recebido aqui não veio da URL.
    // Ele veio do token do usuário autenticado.
    //
    // Por isso chamamos de updateMe.
    // =========================================================

    async updateMe(
        id: number,
        data: {
            name?: string,
            email?: string,
            password?: string
        }
    ) {

        // encontra o usuário pelo id que veio do token
        const user = await repo.findOne({
            where: {
                id
            }
        })

        if (!user) {
            throw new NotFoundError("Usuário não encontrado!")
        }

        // Só vamos alterar/atualizar os campos que vierem
        if (data.name) user.name = data.name
        if (data.email) user.email = data.email

        // Se vier uma senha nova, a gente precisa criptografar ela de novo
        if (data.password) {
            user.password = await bcrypt.hash(data.password, 10)
        }

        // Depois de tudo isso acima, salvamos de novo
        // Como o user já possui id, o TypeORM entende que é atualização, não novo cadastro
        const updatedUser = await repo.save(user)

        // Retorna o usuário sem a senha
        return omitPassword(updatedUser)
    },


    // =========================================================
    // MÉTODO ANTIGO
    //
    // Assim como no update antigo, recebíamos um id que
    // originalmente vinha da URL.
    // =========================================================

    /*
    async delete(id: number) {
        const result = await repo.delete(id)

        if (result.affected === 0) {
            throw new NotFoundError("Usuário não encontrado!")
        }
    }
    */


    // =========================================================
    // NOVO MÉTODO
    //
    // O id vem do token.
    //
    // Portanto, esse método exclui o próprio usuário autenticado.
    // =========================================================

    async deleteMe(id: number) {

        const result = await repo.delete(id)

        if (result.affected === 0) {
            throw new NotFoundError("Usuário não encontrado!")
        }
    }
}
```
### PostService

O `PostService` usa o ID obtido do JWT para associar posts ao usuário e também para verificar propriedade antes de atualizar ou excluir.

**`src\services\PostService.ts`**

```ts
import { AppDataSource } from "../config/data-source"
import { Post } from "../models/Post"
import { User } from "../models/User"
import { NotFoundError } from "./UserService"
import { omitPassword } from "../utils/omitPassword"

const postRepo = AppDataSource.getRepository(Post)
const userRepo = AppDataSource.getRepository(User)

export class ForbiddenError extends Error {}

export const PostService = {

    // Lista todos os posts junto com o autor
    async listAll() {

        const posts = await postRepo.find({
            relations: {
                user: true
            }
        })

        // Remove a senha do usuário antes de retornar
        return posts.map(post => ({
            ...post,
            user: omitPassword(post.user)
        }))
    },


    // Busca um post pelo id
    async getById(id: number) {

        const post = await postRepo.findOne({
            where: {
                id
            },
            relations: {
                user: true
            }
        })

        if (!post) {
            throw new NotFoundError("Post não encontrado!")
        }

        return {
            ...post,
            user: omitPassword(post.user)
        }
    },


    // Cria um post para o usuário autenticado
    async create(
        userId: number,
        data: {
            title: string
            content: string
        }
    ) {

        // O userId vem do JWT validado pelo authMiddleware
        const user = await userRepo.findOne({
            where: {
                id: userId
            }
        })

        if (!user) {
            throw new NotFoundError("Usuário não encontrado!")
        }

        const post = postRepo.create({
            title: data.title,
            content: data.content,
            user
        })

        const savedPost = await postRepo.save(post)

        return {
            ...savedPost,
            user: omitPassword(user)
        }
    },


    // Atualiza um post
    async update(
        postId: number,
        userId: number,
        data: {
            title?: string
            content?: string
        }
    ) {

        const post = await postRepo.findOne({
            where: {
                id: postId
            },
            relations: {
                user: true
            }
        })

        if (!post) {
            throw new NotFoundError("Post não encontrado!")
        }

        // O usuário só pode alterar posts que pertencem a ele
        if (post.user.id !== userId) {
            throw new ForbiddenError(
                "Você não tem permissão para alterar este post!"
            )
        }

        if (data.title !== undefined) {
            post.title = data.title
        }

        if (data.content !== undefined) {
            post.content = data.content
        }

        const updatedPost = await postRepo.save(post)

        return {
            ...updatedPost,
            user: omitPassword(updatedPost.user)
        }
    },


    // Exclui um post
    async delete(postId: number, userId: number) {

        const post = await postRepo.findOne({
            where: {
                id: postId
            },
            relations: {
                user: true
            }
        })

        if (!post) {
            throw new NotFoundError("Post não encontrado!")
        }

        // O usuário só pode excluir posts que pertencem a ele
        if (post.user.id !== userId) {
            throw new ForbiddenError(
                "Você não tem permissão para excluir este post!"
            )
        }

        await postRepo.remove(post)
    }
}
```
---

## Controllers

### AuthController

**`src\controllers\AuthController.ts`**

```ts
import { NextFunction, Request, Response } from "express"
import { UserService } from "../services/UserService"

export class AuthController {

    async login(req: Request, res: Response, next: NextFunction) {
        try {
            const { email, password } = req.body

            // Chamamos o Service para fazer a regra de login
            const result = await UserService.login({
                email,
                password
            })

            // Se deu certo, retornamos usuário sem senha + token
            return res.json(result)

        } catch (error) {
            // Se deu erro, mandamos para o errorHandler
            next(error)
        }
    }
}
```
### UserController

Na atualização e exclusão da própria conta, o ID do usuário vem do JWT.

**`src\controllers\UserController.ts`**

```ts
import { NextFunction, Request, Response } from "express"
import { UserService } from "../services/UserService"

export class UserController {

    async list(req: Request, res: Response, next: NextFunction) {
        try {
            const users = await UserService.listAll()
            return res.json(users)
        } catch (error) {
            next(error)
        }
    }

    async getById(req: Request, res: Response, next: NextFunction) {
        try {
            const id = Number(req.params.id)
            const user = await UserService.getById(id)
            return res.json(user)
        } catch (error) {
            next(error)
        }
    }

    async create(req: Request, res: Response, next: NextFunction) {
        try {
            const { name, email, password } = req.body

            const user = await UserService.create({
                name,
                email,
                password
            })

            return res.status(201).json(user)

        } catch (error) {
            next(error)
        }
    }


    // =========================================================
    // MÉTODO ANTIGO
    // Esse método permitia escolher qual usuário seria atualizado
    // através do id enviado na URL.
    //
    // Isso permitiria, por exemplo:
    //
    // PATCH /users/5
    //
    // Um usuário logado poderia tentar alterar outro usuário
    // simplesmente mudando o id da URL.
    // =========================================================

    /*
    async update(req: Request, res: Response, next: NextFunction) {
        try {
            const id = Number(req.params.id)
            const { name, email, password } = req.body

            const user = await UserService.update(id, { name, email, password })

            return res.json(user)
        } catch (error) {
            next(error)
        }
    }
    */


    // =========================================================
    // NOVO MÉTODO
    //
    // Agora NÃO pegamos mais o id pela URL.
    //
    // O id vem do usuário autenticado.
    // Esse usuário foi colocado dentro do req pelo middleware
    // de autenticação depois que o token foi validado.
    //
    // Dessa forma, o usuário só consegue atualizar a própria conta.
    // =========================================================

    async update(req: Request, res: Response, next: NextFunction) {
        try {

            // Pegamos o id que veio do token
            const id = (req as any).user.id

            const { name, email, password } = req.body

            const user = await UserService.updateMe(
                id,
                {
                    name,
                    email,
                    password
                }
            )

            return res.json(user)

        } catch (error) {
            next(error)
        }
    }


    // =========================================================
    // MÉTODO ANTIGO
    // Recebia o id do usuário pela URL.
    //
    // DELETE /users/5
    //
    // Isso não é adequado para a exclusão da própria conta,
    // pois o usuário poderia trocar o id manualmente.
    // =========================================================

    /*
    async delete(req: Request, res: Response, next: NextFunction) {
        try {
            const id = Number(req.params.id)
            await UserService.delete(id)
            return res.status(204).send()
        } catch (error) {
            next(error)
        }
    }
    */


    // =========================================================
    // NOVO MÉTODO
    //
    // O id do usuário vem do token.
    //
    // Portanto, não precisamos receber nenhum id pela URL.
    //
    // DELETE /users/me
    // =========================================================

    async delete(req: Request, res: Response, next: NextFunction) {
        try {

            // Pegamos o id do próprio usuário autenticado
            const id = (req as any).user.id

            await UserService.deleteMe(id)

            return res.status(204).send()

        } catch (error) {
            next(error)
        }
    }
}
```
### PostController

O ID do post vem da URL, enquanto o ID do usuário autenticado vem do JWT.

**`src\controllers\PostController.ts`**

```ts
import { NextFunction, Request, Response } from "express"
import { PostService } from "../services/PostService"

export class PostController {

    // Lista todos os posts
    async list(req: Request, res: Response, next: NextFunction) {
        try {

            const posts = await PostService.listAll()

            return res.json(posts)

        } catch (error) {
            next(error)
        }
    }


    // Busca um post pelo id
    async getById(req: Request, res: Response, next: NextFunction) {
        try {

            const id = Number(req.params.id)

            const post = await PostService.getById(id)

            return res.json(post)

        } catch (error) {
            next(error)
        }
    }


    // Cria um novo post
    async create(req: Request, res: Response, next: NextFunction) {
        try {

            // Esse id foi colocado no req pelo authMiddleware
            // depois que o JWT foi validado
            const userId = (req as any).user.id

            const { title, content } = req.body

            const post = await PostService.create(
                userId,
                {
                    title,
                    content
                }
            )

            return res.status(201).json(post)

        } catch (error) {
            next(error)
        }
    }


    // Atualiza um post
    async update(req: Request, res: Response, next: NextFunction) {
        try {

            // O id do post vem da URL
            const postId = Number(req.params.id)

            // O id do usuário vem do JWT
            const userId = (req as any).user.id

            const { title, content } = req.body

            const post = await PostService.update(
                postId,
                userId,
                {
                    title,
                    content
                }
            )

            return res.json(post)

        } catch (error) {
            next(error)
        }
    }


    // Exclui um post
    async delete(req: Request, res: Response, next: NextFunction) {
        try {

            const postId = Number(req.params.id)

            // O dono da ação vem do JWT
            const userId = (req as any).user.id

            await PostService.delete(
                postId,
                userId
            )

            return res.status(204).send()

        } catch (error) {
            next(error)
        }
    }
}
```
---

## Middlewares

### authMiddleware

**`src\middlewares\authMiddleware.ts`**

```ts
import { NextFunction, Request, Response } from "express"
import { verifyToken } from "../utils/jwt"

// Middleware para proteger rotas que exigem autenticação
export function authMiddleware(req: Request, res: Response, next: NextFunction) {
    // Pega o header de autorização da requisição
    const authHeader = req.headers.authorization

    // Se não houver header, retorna erro 401
    if (!authHeader) {
        return res.status(401).json({
            message: "Token não fornecido."
        })
    }

    // O token vem neste formato:
    // Authorization: Bearer tokenAqui
    const parts = authHeader.split(" ")

    // Se não tiver exatamente duas partes, está mal formatado
    if (parts.length !== 2) {
        return res.status(401).json({
            message: "Token mal formatado."
        })
    }

    const [scheme, token] = parts

    // A primeira parte precisa ser Bearer
    if (scheme !== "Bearer") {
        return res.status(401).json({
            message: "Formato do token inválido."
        })
    }

    // Verifica se o token é válido
    const decoded = verifyToken(token)

    // Se o token for inválido ou expirado, bloqueia
    if (!decoded) {
        return res.status(401).json({
            message: "Token inválido ou expirado."
        })
    }

    // Guardamos os dados decodificados dentro do req
    // Assim, outros controllers poderiam saber quem é o usuário logado
    (req as any).user = decoded

    // Se chegou até aqui, está tudo certo
    // Então deixamos a requisição seguir
    next()
}
```
### validateUser

**`src\middlewares\validateUser.ts`**

```ts
import { NextFunction, Request, Response } from "express"

// Middleware simples para validar os dados de cadastro de usuário
export function validateUser(req: Request, res: Response, next: NextFunction) {
    const { name, email, password } = req.body

    if (!name || !email || !password) {
        return res.status(400).json({
            message: "Nome, email e senha são obrigatórios."
        })
    }

    if (password.length < 6) {
        return res.status(400).json({
            message: "A senha precisa ter pelo menos 6 caracteres."
        })
    }

    next()
}
```
### validatePost

A criação exige `title` e `content`. Na atualização, basta enviar pelo menos um dos dois campos.

**`src\middlewares\validatePost.ts`**

```ts
import { NextFunction, Request, Response } from "express"

// Validação usada na criação de posts
export function validatePost(
    req: Request,
    res: Response,
    next: NextFunction
) {

    const { title, content } = req.body

    if (!title || !content) {
        return res.status(400).json({
            message: "Título e conteúdo são obrigatórios."
        })
    }

    next()
}


// Validação usada na atualização
// Pelo menos um dos campos precisa ser enviado
export function validatePostUpdate(
    req: Request,
    res: Response,
    next: NextFunction
) {

    const { title, content } = req.body

    if (title === undefined && content === undefined) {
        return res.status(400).json({
            message: "Informe pelo menos título ou conteúdo para atualizar."
        })
    }

    next()
}
```
### errorHandler

Além dos erros já existentes, há tratamento de `403 Forbidden` quando um usuário autenticado tenta alterar ou excluir um post de outra pessoa.

**`src\middlewares\errorHandler.ts`**

```ts
import { NextFunction, Request, Response } from "express"
import {
    NotFoundError,
    UnauthorizedError
} from "../services/UserService"
import { ForbiddenError } from "../services/PostService"

// Esse middleware formata as respostas de erro da aplicação
export function errorHandler(
    error: any,
    req: Request,
    res: Response,
    next: NextFunction
) {

    console.error("Erro capturado pelo errorHandler: ", error)

    // Recurso não encontrado
    if (error instanceof NotFoundError) {
        return res.status(404).json({
            message: error.message
        })
    }

    // Falha de autenticação
    if (error instanceof UnauthorizedError) {
        return res.status(401).json({
            message: error.message
        })
    }

    // Usuário autenticado, mas sem permissão
    if (error instanceof ForbiddenError) {
        return res.status(403).json({
            message: error.message
        })
    }

    // Registro duplicado no MySQL
    if (error.code === "ER_DUP_ENTRY") {
        return res.status(409).json({
            message: "Registro duplicado (email já existente)."
        })
    }

    return res.status(500).json({
        message: "Erro interno do servidor. Traduzindo: DEU RUIM, GURIZADA!"
    })
}
```
---

## Rotas

### Autenticação

**`src\routes\auth.routes.ts`**

```ts
import { Router } from "express"
import { AuthController } from "../controllers/AuthController"

const router = Router()
const authController = new AuthController()

router.post("/login", authController.login.bind(authController))

export default router
```
### Usuários

`PUT /users` e `DELETE /users` atuam sobre o próprio usuário autenticado, identificado pelo JWT.

**`src\routes\user.routes.ts`**

```ts
import { Router } from "express"
import { UserController } from "../controllers/UserController"
import { validateUser } from "../middlewares/validateUser"
import { authMiddleware } from "../middlewares/authMiddleware"

const router = Router()
const userController = new UserController()

// Cadastrar usuário fica público
// Afinal, se a pessoa ainda não tem conta, ela precisa conseguir se cadastrar
router.post("/", validateUser, userController.create.bind(userController))

// Daqui para baixo, as rotas exigem token
router.get("/", authMiddleware, userController.list.bind(userController))
router.get("/:id",authMiddleware, userController.getById.bind(userController))
router.put("/",authMiddleware, userController.update.bind(userController))
router.delete("/",authMiddleware, userController.delete.bind(userController))

export default router
```
### Posts

As rotas de leitura são públicas. Criar, atualizar e excluir exigem JWT.

**`src\routes\post.routes.ts`**

```ts
import { Router } from "express"
import { PostController } from "../controllers/PostController"
import { authMiddleware } from "../middlewares/authMiddleware"
import {
    validatePost,
    validatePostUpdate
} from "../middlewares/validatePost"

const router = Router()
const postController = new PostController()


// Listar todos os posts
router.get(
    "/",
    postController.list.bind(postController)
)


// Buscar um post pelo id
router.get(
    "/:id",
    postController.getById.bind(postController)
)


// Criar post
// Precisa estar autenticado
router.post(
    "/",
    authMiddleware,
    validatePost,
    postController.create.bind(postController)
)


// Atualizar post
// O JWT identifica o usuário
// O Service verifica se ele é o dono do post
router.put(
    "/:id",
    authMiddleware,
    validatePostUpdate,
    postController.update.bind(postController)
)


// Excluir post
// O JWT identifica o usuário
// O Service verifica se ele é o dono do post
router.delete(
    "/:id",
    authMiddleware,
    postController.delete.bind(postController)
)


export default router
```
### Agrupamento das rotas

**`src\routes\index.ts`**

```ts
import { Router } from "express"
import userRoutes from "./user.routes"
import authRoutes from "./auth.routes"
import postRoutes from "./post.routes"

const router = Router()

router.use("/users", userRoutes)
router.use("/auth", authRoutes)
router.use("/posts", postRoutes)

export default router
```
---

## Servidor

**`src\server.ts`**

```ts
import "reflect-metadata"
import "dotenv/config"
import express from "express"
import cors from "cors"
import { AppDataSource } from "./config/data-source"
import routes from "./routes"
import { errorHandler } from "./middlewares/errorHandler"

const app = express()

app.use(cors())
app.use(express.json())

app.use(routes)

// O errorHandler precisa ser o ÚLTIMO middleware, depois de todas as rotas
app.use(errorHandler)

const PORT = process.env.PORT || 3000

AppDataSource.initialize()
    .then(() => {
        console.log("Conexão com o banco de dados estabelecida.")

        app.listen(PORT, () => {
            console.log(`Servidor rodando em http://localhost:${PORT}`)
        })
    })
    .catch((error) => {
        console.error("Erro ao conectar com o banco de dados:", error)
    })
```
---

## 🧪 Rotas finais para testar

| Método | Rota | JWT | Descrição |
|---|---|---:|---|
| `POST` | `/users` | Não | Cadastrar usuário |
| `POST` | `/auth/login` | Não | Fazer login e receber token |
| `GET` | `/users` | Sim | Listar usuários |
| `GET` | `/users/:id` | Sim | Buscar usuário |
| `PUT` | `/users` | Sim | Atualizar a própria conta |
| `DELETE` | `/users` | Sim | Excluir a própria conta |
| `GET` | `/posts` | Não | Listar posts |
| `GET` | `/posts/:id` | Não | Buscar post |
| `POST` | `/posts` | Sim | Criar post para o usuário autenticado |
| `PUT` | `/posts/:id` | Sim | Atualizar um post próprio |
| `DELETE` | `/posts/:id` | Sim | Excluir um post próprio |

### Header das rotas protegidas

```http
Authorization: Bearer SEU_TOKEN_AQUI
```

### Exemplo de cadastro

```http
POST /users
Content-Type: application/json
```

```json
{
  "name": "Leonardo",
  "email": "leo@email.com",
  "password": "123456"
}
```

### Exemplo de login

```http
POST /auth/login
Content-Type: application/json
```

```json
{
  "email": "leo@email.com",
  "password": "123456"
}
```

### Exemplo de criação de post

O cliente não envia `userId`. O backend descobre o autor usando o JWT.

```http
POST /posts
Authorization: Bearer SEU_TOKEN_AQUI
Content-Type: application/json
```

```json
{
  "title": "Meu primeiro post",
  "content": "Conteúdo do post"
}
```

### Exemplo de atualização de post

```http
PUT /posts/1
Authorization: Bearer SEU_TOKEN_AQUI
Content-Type: application/json
```

```json
{
  "title": "Título atualizado"
}
```

### Exemplo de exclusão

```http
DELETE /posts/1
Authorization: Bearer SEU_TOKEN_AQUI
```

Se o post não pertencer ao usuário autenticado, a API retorna `403 Forbidden`.
