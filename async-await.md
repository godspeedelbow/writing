# Wait up for `async/await`, it's great

`async/await` is a new javascript feature that allows you to write asynchronous code without callbacks. This makes your code easier to read and easier to reason about. 

It works like this: whenever you use a Promise to deal with asynchronicity, use the `await` keyword to 'halt' the code execution cursor until the Promise is resolved.

*old goodness:*
```js
getUserPromise(userId).then(user => console.log('user', user)); // execute the callback when Promise resolves
```

*new goodness:*
```js
const user = await getUserPromise(userId); // put result in `user` when Promise resolves
console.log('user', user); // continue sychronously as if nothing happened
```

As you don't need to use callbacks anymore to deal with asynchronicity, you can now write your code _as if all operations are synchronous_ which is A Good Thing™:

- **'sychronous' execution** is easier to comprehend
- **less functions** means less boiler plate and less indentation
- a **single, shared function scope** removes the need to 'pass data around' - _it's just there_

## A non-trivial example

The power of `async/await` truly shines in situations where there are multiple asynchronous operations that need to be composed in to a single result.

Consider the following functionality:
- fetch a user and their posts (in parallel)
- get the comments on those posts
- parse the posts with the comments
- log the user and the parsed posts

You can implement the above functionality with async/await quite elegantly as follows.

### Promises - using async-await

```js
const run = async (userId) => {
  const [user, posts] = await Promise.all([
    getUser(userId),
    getPosts(userId),
  ]);

  const comments = await getComments(posts);
  const postsWithComments = await parsePostsWithComments(posts, comments);

  console.log('user', user);
  console.log('postsWithComments', postsWithComments);
}

try {
  run(userId);
} catch (err) {
  console.log('err', err);
}
```

**Note:**
- whenever you use `await`, the surrounding function needs be declared te be `async`
- use `try/catch` to handle any Promise that fails within the `async` function
- complete error stack, because we use Promises

Looks quite neat, huh? Easy to read and easy to reason about! Now consider how this code would look if you don't use `async/await`.

### Promises, without async/await

```js
const run = (userId) => {
  Promise.all([
    getUser(userId),
    getPosts(userId),
  ]).then([user, posts, comments] => {
    getComments(posts).then(comments => {
      parsePostsWithComments(posts, comments).then(postsWithComments => {
        console.log('user', user);
        console.log('postsWithComments', postsWithComments);
      });
    });
  }).fail(err => {
    console.log('err', err);
  });
}
```

**Note:**
- there's a need for nested Promises because earlier data (e.g. `user`) needs to be accessible to be used later (i.e. when logging it)
- you can also store data in the outer function scope of `run`. Then you don't need to nest Promises, as they all have access to the data via the scope of `run`. You can also use a Promise library to store the data on a context object so that it is available in subsequent Promises (usually via `this`). However, both solutions don't scale well as functionality increases in complexity

### `async` module

```js
const run = (userId) => {
  async.waterfall([
    callback => {
      async.parallel([
        cb => getUser(userId, cb),
        cb => getPosts(userId, cb),
      ], callback);
    },
    ([user, posts], callback) => {
      getComments(posts, (err, comments) => callback(err, user, posts, comments));
    },
    (user, posts, comments, callback) => {
      parsePostsWithComments(posts, comments, (err, postsWithComments) => {
        callback(err, user, postsWithComments);
      });
    },
  ], (err, user, postsWithComments) => {
    if (err) {
      console.log('err', err);
    } else 
      console.log('user', user);
      console.log('postsWithComments', postsWithComments);
  });
}
```

**Note:**
- 'passing along data' is cumbersome and so is error-handling (`if (err) return done(err);` much lately?)
- because of the nature of callbacks in nodejs, errors stacks are incomplete
- there are other control flow libraries that offer similar solutions

### Mere callbacks

```js
const run = (userId) => {
  let user;
  let posts;

  getUser(user, (err, _user) => {
    if (err) return done(err);
    user = _user;
    if (posts) pleaseContinue(); // `posts` already fetched? continue
  });
  getPosts(posts, (err, _posts) => {
    if (err) return done(err);
    posts = _posts;
    if (user) pleaseContinue(); // `user` already fetched? continue
  });

  const pleaseContinue = () => {
    getComments(posts, (err, comments) => {
      if (err) return done(err);

      parsePostsWithComments(posts, comments, (err, postsWithComments) => {
        done(err, user, postsWithComments)
      });
    });
  }
}

const done = (err, postsWithComments) => {
  if (err) {
    console.log('err', err);
  } else 
    console.log('user', user);
    console.log('postsWithComments', postsWithComments);
  }
};
```

**Note:**
- getting `getUser` and `getPosts` to execute in parallel, is pretty ugly without a proper abstraction

## Why wait?

With `async/await`, you can write code that is easier to read and reason about. Why wait?

It's native Javascript, so it's a matter of time before it's widely supported. To start using it today, use node 7 with `--harmony`. It's also supported by Chrome (link tweet) and in any case, you can always use `babel` to transpile it into regular ol' javascript.
