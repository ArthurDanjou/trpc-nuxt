## Authorization

The `createContext`-function is called for each incoming request so here you can add contextual information about the calling user from the request object.

### Create context from request headers

```ts
// ~/server/trpc/index.ts
import type * as trpc from '@trpc/server'
import type { CompatibilityEvent } from 'h3'
import { decodeAndVerifyJwtToken } from '~/somewhere/in/your/app/utils'

// The app's context - is generated for each incoming request
export async function createContext({ req }: CompatibilityEvent) {
  // Create your context based on the request object
  // Will be available as `ctx` in all your resolvers

  // This is just an example of something you'd might want to do in your ctx fn
  async function getUserFromHeader() {
    if (req.headers.authorization) {
      const user = await decodeAndVerifyJwtToken(req.headers.authorization.split(' ')[1])
      return user
    }
    return null
  }
  const user = await getUserFromHeader()

  return {
    user,
  }
}

type Context = trpc.inferAsyncReturnType<typeof createContext>

// [..] Define API handler and app router
```

### Option 1: Authorize using resolver

```ts
export const router = trpc
  .router<Context>()
  // open for anyone
  .query('hello', {
    input: z.string().nullish(),
    resolve: ({ input, ctx }) => {
      return `hello ${input ?? ctx.user?.name ?? 'world'}`
    },
  })
  // checked in resolver
  .query('secret', {
    resolve: ({ ctx }) => {
      if (!ctx.user)
        throw new trpc.TRPCError({ code: 'UNAUTHORIZED' })

      return {
        secret: 'sauce',
      }
    },
  })
```

### Option 2: Authorize using middleware

```ts
import * as trpc from '@trpc/server'
import { TRPCError } from '@trpc/server'
import { createRouter } from '../createRouter'

export const router = trpc
  .router<Context>()
  // this is accessible for everyone
  .query('hello', {
    input: z.string().nullish(),
    resolve: ({ input, ctx }) => {
      return `hello ${input ?? ctx.user?.name ?? 'world'}`
    },
  })
  .merge(
    'admin.',
    createRouter()
      // this protects all procedures defined next in this router
      .middleware(async ({ ctx, next }) => {
        if (!ctx.user?.isAdmin)
          throw new TRPCError({ code: 'UNAUTHORIZED' })

        return next()
      })
      .query('secret', {
        resolve: ({ ctx }) => {
          return {
            secret: 'sauce',
          }
        },
      }),
  )
```

Learn more about authorization [here](https://trpc.io/docs/authorization).
