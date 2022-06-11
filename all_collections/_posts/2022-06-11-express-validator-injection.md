---
layout: post
title: Middleware Injection using express-validator and Inversify
date: 2022-06-11
categories: ["node", "typescript"]
---

I've recently searched the internet for a 'clean' way of using express-validator with Inversify. Unfortunately I didn't find anything that covered my use case, so I had to come with something myself. I hope that whomever stumbles upon this post will find it helpful in some way. Initially I've started with code that somewhat resembled this:

```ts
@controller('/api/v1/posts')
export class PostsController extends BaseHttpController {
  constructor() {
    super();
  }

  @httpGet('/:id', [params("id").toInt()])
  public async getById(
    @requestParam('id') id: number,
  ): Promise<IHttpActionResult> {
    // return a post
}
```

This however left me unsatisfied for a number of reasons. Firstly, the controller has more than _one reason to change_, since we have to make changes to this class if either the request handling needs to change _or_ changes to the validation chain are necessary. It therefore violates the single responsibility principle.
Furthermore, I felt like the point of using an IoC Container like Inversify in the first place, is to actually - well - inject your dependencies into your classes.

Since input validation is something that should happen on almost every API endpoint, I figured it would be nice to have all the validations for an endpoint contained in a single class, that would be easy to understand and even easier to test.

To achieve this, we can create an abstract _BaseValidationMiddleware_ class, that will handle the evaluation of the validations. Also an abstract _validate_ function is defined, which we will use in a minute.

```ts
export abstract class BaseValidationMiddleware extends BaseMiddleware {
  public async handler(
    req: Request,
    res: Response,
    next: NextFunction
  ): Promise<void> {
    await this.validate(req);
    this.handleErrors(req, res, next);
  }

  protected abstract validate(req: Request): Promise<void>;

  private handleErrors(
    req: Request,
    res: Response,
    next: NextFunction
  ): void | Response {
    const errors = validationResult(req);

    return errors.isEmpty()
      ? next()
      : res.status(400).json({ errors: errors.array() });
  }
}
```

Now we can derive a route specific validation class, that defines the express-validator validation chains and runs them imperatively. Notice that the results will be handled in the _handleErrors_ function of the parent class.

```ts
@injectable()
export class RouteValidationMiddleware extends BaseValidationMiddleware {
  protected async validate(req: Request): Promise<void> {
    const validations = [params("id").toInt()];

    await Promise.all(validations.map((validation) => validation.run(req)));
  }
}
```

The class also needs to be registered in a fashion similar to the following.

```ts
const TYPES = {
  RouteValidation: Symbol.for("RouteValidation"),
};

const container = new Container();

container
  .bind<RouteValidationMiddleware>(TYPES.RouteValidation)
  .to(RouteValidationMiddleware);

export { TYPES, container };
```

Using this approach it is very easy to inject the middleware into the controller, because the httpGet decorator provided by the inversify-express-utils will just accept the symbols for a certain registration.

```ts
@controller('/api/v1/posts')
export class PostsController extends BaseHttpController {
  constructor() {
    super();
  }

  @httpGet('/:id', TYPES.RouteValidation)
  public async getById(
    @requestParam('id') id: number,
  ): Promise<IHttpActionResult> {
    // return a post
}
```

#### References

1. [Express Validator Docs](https://express-validator.github.io/docs/)
2. [InversifyJS Docs](https://inversify.io/)
3. [inversify-express-utils examples repo](https://github.com/inversify/inversify-express-example)
