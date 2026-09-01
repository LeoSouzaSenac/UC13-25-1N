# JWT Backend (sem camada Repository) — Código completo
Backend em Node.js + TypeScript com TypeORM (MySQL), Express e autenticação JWT.
Arquitetura: **Model → Service → Controller → Routes**.

---

## 📦 Instalação

1. Instale as dependências:

```bash
npm install
```

2. Se quiser instalar do zero (sem o `package.json` pronto), os comandos seriam:

```bash
npm init -y
npm install express cors dotenv typeorm mysql2 reflect-metadata bcrypt jsonwebtoken
npm install --save-dev typescript ts-node-dev @types/node @types/express @types/cors @types/bcrypt @types/jsonwebtoken
```

3. Crie o banco de dados no MySQL (nome deve bater com `DB_DATABASE` do `.env`):

```sql
CREATE DATABASE jwt_backend;
```

4. Suba o servidor em modo desenvolvimento:

```bash
npm run dev
```

---

## 📁 Estrutura de pastas

```
src/
  config/data-source.ts
  models/User.ts, Post.ts
  services/UserService.ts
  controllers/UserController.ts, AuthController.ts
  middlewares/authMiddleware.ts, validateUser.ts, errorHandler.ts
  routes/user.routes.ts, auth.routes.ts, index.ts
  utils/jwt.ts, omitPassword.ts
  server.ts
```

---

## 📄 Arquivos

### Instalação e configuração

**`package.json`**

```json
{
  "name": "jwt-backend",
  "version": "1.0.0",
  "description": "Backend com TypeORM, Express e autenticação JWT (Repository/Service/Controller/Middleware)",
  "main": "src/server.ts",
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js"
  },
  "dependencies": {
    "bcrypt": "^5.1.1",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "jsonwebtoken": "^9.0.2",
    "mysql2": "^3.11.0",
    "reflect-metadata": "^0.2.2",
    "typeorm": "^0.3.20"
  },
  "devDependencies": {
    "@types/bcrypt": "^5.0.2",
    "@types/cors": "^2.8.17",
    "@types/express": "^4.17.21",
    "@types/jsonwebtoken": "^9.0.6",
    "@types/node": "^20.14.9",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.5.3"
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

```
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
JWT_EXPIRES_IN=1d

```

**`.gitignore`**

```text
node_modules
dist
.env

```

### Conexão com o banco (TypeORM)

**`src/config/data-source.ts`**

```ts
import "reflect-metadata"
import { DataSource } from "typeorm"
import { User } from "../models/User"
import { Post } from "../models/Post"

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

### Models

**`src/models/User.ts`**

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

**`src/models/Post.ts`**

```ts
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne } from "typeorm"
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
    @ManyToOne(() => User, (user) => user.posts)
    user!: User
}

```

### Utils

**`src/utils/jwt.ts`**

```ts
import jwt from "jsonwebtoken"

interface Payload {
    id: number
    email: string
}

export function generateToken(payload: Payload) {
    return jwt.sign(payload, process.env.JWT_SECRET!, {
        expiresIn: process.env.JWT_EXPIRES_IN
    })
}

export function verifyToken(token: string) {
    try {
        return jwt.verify(token, process.env.JWT_SECRET!)
    } catch {
        return null
    }
}

```

**`src/utils/omitPassword.ts`**

```ts
import { User } from "../models/User"

// Remove a senha do objeto de usuário antes de retornar para o cliente
export function omitPassword(user: User) {
    const { password, ...userWithoutPassword } = user
    return userWithoutPassword
}

```

### Service (sem repository)

**`src/services/UserService.ts`**

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
        const users = await repo.find({ relations: ["posts"] })
        return users.map(user => omitPassword(user))
    },

    async getById(id: number) {
        const user = await repo.findOne({ where: { id }, relations: ["posts"] })

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
        const user = await repo.findOne({ where: { email: data.email } })

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

    async update(id: number, data: { name?: string, email?: string, password?: string }) {
        // encontra o usuário pelo id
        const user = await repo.findOne({ where: { id } })

        if (!user) {
            throw new NotFoundError("Usuário não encontrado!")
        }

        // Só vamos alterar/atualizar os campos que vierem
        if (data.name) user.name = data.name
        if (data.email) user.email = data.email

        // Se vier uma senha nova, a gente precisa criptografar ela de novo
        if (data.password) user.password = await bcrypt.hash(data.password, 10)

        // Depois de tudo isso acima, salvamos de novo
        // Como o user já possui id, o TypeORM entende que é atualização, não novo cadastro
        const updatedUser = await repo.save(user)

        // Retorna o usuário sem a senha
        return omitPassword(updatedUser)
    },

    async delete(id: number) {
        const result = await repo.delete(id)

        if (result.affected === 0) {
            throw new NotFoundError("Usuário não encontrado!")
        }
    }
}

```

### Controllers

**`src/controllers/UserController.ts`**

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

    async delete(req: Request, res: Response, next: NextFunction) {
        try {
            const id = Number(req.params.id)
            await UserService.delete(id)
            return res.status(204).send()
        } catch (error) {
            next(error)
        }
    }
}

```

**`src/controllers/AuthController.ts`**

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

### Middlewares

**`src/middlewares/validateUser.ts`**

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

**`src/middlewares/authMiddleware.ts`**

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
    ;(req as any).user = decoded

    // Se chegou até aqui, está tudo certo
    // Então deixamos a requisição seguir
    next()
}

```

**`src/middlewares/errorHandler.ts`**

```ts
import { NextFunction, Request, Response } from "express"
import { NotFoundError, UnauthorizedError } from "../services/UserService"

// Esse middleware vai formatar cada resposta de erro.
// Ao invés de cada controller ter que pegar um erro e formatar a mensagem bonitinha, ele faz isso pra todo mundo.
export function errorHandler(error: any, req: Request, res: Response, next: NextFunction) {

    // Antes de mais nada, a gente mostra o erro "na forma original" dele pra debugar
    console.error("Erro capturado pelo errorHandler: ", error)

    // Erro para quando alguma coisa não foi encontrada
    if (error instanceof NotFoundError) {
        return res.status(404).json({
            message: error.message
        })
    }

    // Erro para quando o usuário não tem autorização
    // Exemplo: senha inválida
    if (error instanceof UnauthorizedError) {
        return res.status(401).json({
            message: error.message
        })
    }

    // Esse tal de 'ER_DUP_ENTRY' é específico do MySQL:
    // ele acontece quando a gente tenta salvar algo que já existe e tem UNIQUE
    // exemplo: criar um usuário com um email que já existe
    if (error.code === "ER_DUP_ENTRY") {
        return res.status(409).json({
            message: "Registro duplicado (email já existente)."
        })
    }

    // Se for qualquer outro erro que a gente não previu, vira um 500 genérico
    return res.status(500).json({
        message: "Erro interno do servidor. Traduzindo: DEU RUIM, GURIZADA!"
    })
}

```

### Rotas

**`src/routes/user.routes.ts`**

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
router.get("/:id", authMiddleware, userController.getById.bind(userController))
router.put("/:id", authMiddleware, userController.update.bind(userController))
router.delete("/:id", authMiddleware, userController.delete.bind(userController))

export default router

```

**`src/routes/auth.routes.ts`**

```ts
import { Router } from "express"
import { AuthController } from "../controllers/AuthController"

const router = Router()
const authController = new AuthController()

router.post("/login", authController.login.bind(authController))

export default router

```

**`src/routes/index.ts`**

```ts
import { Router } from "express"
import userRoutes from "./user.routes"
import authRoutes from "./auth.routes"

const router = Router()

router.use("/users", userRoutes)
router.use("/auth", authRoutes)

export default router

```

### Servidor

**`src/server.ts`**

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
