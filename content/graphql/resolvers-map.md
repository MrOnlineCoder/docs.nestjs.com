### Resolvers

Typically, you create a resolvers map manually. The `@nestjs/graphql` package, on the other hand, generates a resolvers map automatically using the metadata provided by the decorators. To demonstrate the library basics, we'll create a simple authors API.

#### Schema first

As mentioned in the [previous](/graphql/quick-start) chapter, in the schema first approach we manually define our types in SDL (read [more](http://graphql.org/learn/schema/#type-language)).

```java
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String!
  votes: Int
}

type Query {
  author(id: Int!): Author
}
```

Our GraphQL schema contains a single exposed query - `author(id: Int!): Author`. Now, let's create an `AuthorResolver`.

```typescript
@Resolver('Author')
export class AuthorResolver {
  constructor(
    private readonly authorsService: AuthorsService,
    private readonly postsService: PostsService,
  ) {}

  @Query()
  async author(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveProperty()
  async posts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **Hint** If you use the `@Resolver()` decorator, you don't have to mark a class as an `@Injectable()`.

The `@Resolver()` decorator does not affect queries and mutations (neither `@Query()` nor `@Mutation()` decorators). It only informs Nest that each `@ResolveProperty()` inside this particular class has a parent, which is an `Author` type in this case (`Author.posts` relation). Basically, instead of setting `@Resolver()` at the top of the class, this can be done close to the method:

```typescript
@Resolver('Author')
@ResolveProperty()
async posts(@Parent() author) {
  const { id } = author;
  return this.postsService.findAll({ authorId: id });
}
```

However, if you have multiple `@ResolveProperty()` decorators inside one class, you must add `@Resolver()` to all of them, which is not necessarily a good practice (as it creates extra overhead).

Conventionally, we would use something like `getAuthor()` or `getPosts()` as method names. We can easily do this by passing the real names as arguments of the decorator.

```typescript
@Resolver('Author')
export class AuthorResolver {
  constructor(
    private readonly authorsService: AuthorsService,
    private readonly postsService: PostsService,
  ) {}

  @Query('author')
  async getAuthor(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveProperty('posts')
  async getPosts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **Hint** The `@Resolver()` decorator can be used at the method-level as well.

#### Typings

Assuming that we have enabled the typings generation feature (with `outputAs: 'class'` as shown in the [previous](/graphql/quick-start) chapter), once you run the application it should generate the following file:

```typescript
export class Author {
  id: number;
  firstName?: string;
  lastName?: string;
  posts?: Post[];
}

export class Post {
  id: number;
  title: string;
  votes?: number;
}

export abstract class IQuery {
  abstract author(id: number): Author | Promise<Author>;
}
```

Generating classes (instead of interfaces) allows you to use **decorators**, which makes them extremely useful for validation purposes (read [more](/techniques/validation)). For example:

```typescript
import { MinLength, MaxLength } from 'class-validator';

export class CreatePostInput {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

> warning **Notice** To enable auto-validation of your inputs (and parameters), use `ValidationPipe`. Read more about validation [here](/techniques/validation) or more specifically about pipes [here](/pipes).

However, if you add decorators directly to the automatically generated file, they will be **overwritten** each time the file is generated. Instead, create a separate file and simply extend the generated class.

```typescript
import { MinLength, MaxLength } from 'class-validator';
import { Post } from '../../graphql.ts';

export class CreatePostInput extends Post {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

#### Code first

In the code first approach, we don't write SDL by hand. Instead we use decorators.

```typescript
import { Field, Int, ObjectType } from 'type-graphql';
import { Post } from './post';

@ObjectType()
export class Author {
  @Field(type => Int)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

`Author` model has been created. Now, let's create the missing `Post` class.

```typescript
import { Field, Int, ObjectType } from 'type-graphql';

@ObjectType()
export class Post {
  @Field(type => Int)
  id: number;

  @Field()
  title: string;

  @Field(type => Int, { nullable: true })
  votes?: number;
}
```

With our models in place, we can move to the resolver class.

```typescript
@Resolver(of => Author)
export class AuthorResolver {
  constructor(
    private readonly authorsService: AuthorsService,
    private readonly postsService: PostsService,
  ) {}

  @Query(returns => Author)
  async author(@Args({ name: 'id', type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveProperty()
  async posts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

Conventionally, we would use something like `getAuthor()` or `getPosts()` as method names. We can easily do this by passing the real names as arguments of the decorator.

```typescript
@Resolver(of => Author)
export class AuthorResolver {
  constructor(
    private readonly authorsService: AuthorsService,
    private readonly postsService: PostsService,
  ) {}

  @Query(returns => Author, { name: 'author' })
  async getAuthor(@Args({ name: 'id', type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveProperty('posts', returns => [Post])
  async getPosts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

Usually, you won't have to pass such an object into the `@Args()` decorator. For example, if your identifier's type is string, the following construction would be sufficient:

```typescript
@Args('id') id: string
```

However, the `number` type doesn't give `type-graphql` enough information about the expected GraphQL representation (`Int` vs. `Float`). Thus we have to **explicitly** pass the type reference.

Moreover, you can create a dedicated `AuthorArgs` class:

```typescript
@Args() args: AuthorArgs
```

With the following body:

```typescript
@ArgsType()
class AuthorArgs {
  @Field(type => Int)
  @Min(1)
  id: number;
}
```

> info **Hint** `Int` and both decorators `@Field()` and `@ArgsType()` are imported from the `type-graphql` package, while `@Min()` comes from the `class-validator`.

You may also notice that such classes play very well with the `ValidationPipe` (read [more](/techniques/validation)).

#### Decorators

You may note that we refer to the following arguments using dedicated decorators. Below is a comparison of the provided decorators and the plain Apollo parameters they represent.

<table>
  <tbody>
    <tr>
      <td><code>@Root()</code> and <code>@Parent()</code></td>
      <td><code>root</code>/<code>parent</code></td>
    </tr>
    <tr>
      <td><code>@Context(param?: string)</code></td>
      <td><code>context</code> / <code>context[param]</code></td>
    </tr>
    <tr>
      <td><code>@Info(param?: string)</code></td>
      <td><code>info</code> / <code>info[param]</code></td>
    </tr>
    <tr>
      <td><code>@Args(param?: string)</code></td>
      <td><code>args</code> / <code>args[param]</code></td>
    </tr>
  </tbody>
</table>

#### Module

Once we're done here, we have to provide the `AuthorResolver` somewhere. For example, we can do this in the newly created `AuthorsModule`.

```typescript
@Module({
  imports: [PostsModule],
  providers: [AuthorsService, AuthorResolver],
})
export class AuthorsModule {}
```

The `GraphQLModule` will take care of reflecting the metadata and transforming classes into the correct resolvers map automatically. The only thing you need to be aware of is that you need to import this module somewhere, so Nest will be able to utilize `AuthorsModule`.

> info **Hint** Learn more about GraphQL queries [here](http://graphql.org/learn/queries/).
