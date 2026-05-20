# Fastify Swagger Auth Template

Copy this code to `src/plugins/swagger.ts`. Requires `@fastify/swagger`, `@fastify/swagger-ui`.

Env vars: `TM_EXT_SWAGGER_USER`, `TM_EXT_SWAGGER_PASSWORD`

```typescript
import fp from 'fastify-plugin';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';
import type { FastifyInstance, FastifyRequest, FastifyReply } from 'fastify';

function verifySwaggerAuth(request: FastifyRequest, reply: FastifyReply, done: () => void) {
  const authHeader = request.headers.authorization;
  if (!authHeader || !authHeader.startsWith('Basic ')) {
    reply
      .code(401)
      .header('WWW-Authenticate', 'Basic realm="Swagger"')
      .send({ error: 'Authentication required' });
    return;
  }

  const credentials = Buffer.from(authHeader.slice(6), 'base64').toString();
  const [user, pass] = credentials.split(':');

  const expectedUser = process.env.TM_EXT_SWAGGER_USER;
  const expectedPass = process.env.TM_EXT_SWAGGER_PASSWORD;

  if (!expectedUser || !expectedPass || user !== expectedUser || pass !== expectedPass) {
    reply.code(401).header('WWW-Authenticate', 'Basic realm="Swagger"').send({ error: 'Invalid credentials' });
    return;
  }

  done();
}

export default fp(async (app: FastifyInstance) => {
  await app.register(swagger, {
    openapi: {
      info: {
        title: 'ThriveHR Service API',
        version: '1.0.0',
      },
    },
  });

  await app.register(swaggerUi, {
    routePrefix: '/documentation',
    uiConfig: { docExpansion: 'list', deepLinking: false },
    uiHooks: {
      onRequest: verifySwaggerAuth,
    },
    staticCSP: true,
  });

  app.addHook('onRequest', async (request, reply) => {
    if (request.url === '/documentation/json') {
      const authHeader = request.headers.authorization;
      if (!authHeader?.startsWith('Basic ')) {
        reply.code(401).header('WWW-Authenticate', 'Basic realm="Swagger"').send({ error: 'Authentication required' });
        return;
      }
      const credentials = Buffer.from(authHeader.slice(6), 'base64').toString();
      const [user, pass] = credentials.split(':');
      if (user !== process.env.TM_EXT_SWAGGER_USER || pass !== process.env.TM_EXT_SWAGGER_PASSWORD) {
        reply.code(401).send({ error: 'Invalid credentials' });
        return;
      }
    }
  });
});
```
