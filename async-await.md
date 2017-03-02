# Wait up for `async/await`, it's great!

`async/await` is a new javascript feature that allows you to wrap asynchronous code, so it appears to be synchronous. This makes your code easier to read and easier to reason about.

*callbacks:*
```js
getUser(userId, (err, user) => {
  if (err) return console.error('error', err);
  console.log('user', user);
});
```

*Promises:*
```js
getUserPromise(userId)
  .then(user => console.log('user', user));
  .catch(err => console.error('error', err));
```

*Promises with async/await:*
```js
const user = await getUserPromise(userId); // 'await' the Promise to resolve and assign result to `user`
console.log('user', user);
```

The `await` keyword halts the code execution cursor until the Promise is resolved. As you don't need to use callback functions anymore to deal with asynchronicity, you can now write your code _as if all operations are synchronous_ which is A Good Thing™:

- **'synchronous' (downward) execution** is easier to read and therefore comprehend
- **no callback functions** means less boiler plate and less indentation
- a **single, shared function scope** removes the need to 'pass data around' - _it's just there_

## A trivial example

The power of `async/await` truly shines in situations where there are *multiple asynchronous operations that need to be composed* in to a single result.

Consider the following functionality:
- fetch a user and their posts (in parallel, to speed up execution)
- get the comments on those posts
- parse the posts with the comments

You can implement the above functionality with `async/await` quite elegantly as follows.

### Promises, using async-await

```js
const run = async userId => {
  const [user, posts] = await Promise.all([
    getUser(userId),
    getPosts(userId),
  ]);

  const comments = await getComments(posts);
  return await parsePostsWithComments(user, posts, comments);
}

try {
  console.log('postsWithComments', run(userId);
} catch (err) {
  console.log('err', err);
}
```

**Note:**
- whenever you use `await`, the surrounding function needs to be declared to be `async`
- use `try/catch` to handle any Promise that fails within the `async` function

Looks quite neat, huh? Below I show how this could be implemented without `async/await` with only Promises/callbacks. What you see is that the code has more boiler plate, is more indented and therefore harder to read.

The reason for this is that although the example is trivial, data needs to be passed around, by either:
- storing the data in the shared, outer function scope, or;
- passing the data along from one [`async`](http://caolan.github.io/async/).waterfall callback to the next

Take a look for yourself!

### Promises, without async/await

```js
const run = (userId) => {
  Promise.all([
    getUser(userId),
    getPosts(userId),
  ]).then([user, posts, comments] => {
    getComments(posts).then(comments => {
      parsePostsWithComments(user, posts, comments).then(postsWithComments => {
        console.log('postsWithComments', postsWithComments);
      });
    });
  }).catch(err => {
    console.log('err', err);
  });
}
```

**Note:**
- Promises are nested because earlier data (ie. `user` and `posts`) needs to be accessible to be used later

### Callbacks with [`async` module](http://caolan.github.io/async/)

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
      parsePostsWithComments(user, posts, comments, callback);
    },
  ], (err, postsWithComments) => {
    if (err) {
      console.log('err', err);
    } else
      console.log('postsWithComments', postsWithComments);
  });
}
```

**Note:**
- 'passing along data' is cumbersome and so is error-handling (`if (err) return done(err);` much lately?)
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

      parsePostsWithComments(user, posts, comments, done);
    });
  }
}

const done = (err, postsWithComments) => {
  if (err) {
    console.log('err', err);
  } else
    console.log('postsWithComments', postsWithComments);
  }
};
```

**Note:**
- as you see, getting `getUser` and `getPosts` to execute in parallel is pretty ugly without a proper abstraction like [`async`](http://caolan.github.io/async/).parallel or a similar library

## Why wait?

With `async/await`, you can write code that is easier to read and reason about. It's native Javascript so it's a matter of time before it's widely supported. To start using it today, use node 7 with `--harmony`. It's also [supported by Chrome](https://twitter.com/addyosmani/status/789126892402204673) and in any case, you can always use `babel` to transpile it into regular ol' javascript.
