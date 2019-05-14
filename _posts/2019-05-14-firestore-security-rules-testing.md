---
layout: post
title: Getting started with firestore security rules testing (firestore emulator + Jest + CircleCi)
---

This article covers the setup process for a basic project using the firestore
emulator, `@firebase/testing`, Jest and CircleCi for unit testing of firestore
security rules.

The final project on github:
[JakeHedman/firestore-security-rules-testing](https://github.com/JakeHedman/firestore-security-rules-testing)

## Firebase project setup

Use
[firebase-tools](https://firebase.google.com/docs/cli#initialize_a_firebase_project)
to create your project with firestore enabled if you haven't already.

```sh
npm install -g firebase-tools
firebase login
mkdir my-project
cd my-project
firebase init
```

[Git commit](https://github.com/JakeHedman/firestore-security-rules-testing/commit/firebase-project-setup)

## Moving firestore 

I like to keep my firestore files in a subdirectory. Start by moving rules and
indexes:

```sh
mkdir firestore
mv firestore.rules firestore.indexes.json firestore
```

And update `firebase.json`:

```diff
 {
   "firestore": {
-    "rules": "firestore.rules",
-    "indexes": "firestore.indexes.json"
+    "rules": "firestore/firestore.rules",
+    "indexes": "firestore/firestore.indexes.json"
   }
 }
```

[Git commit](https://github.com/JakeHedman/firestore-security-rules-testing/commit/moving-firestore)

## Security rules

Add some security rule to test in `firestore/firestore.rules`

```diff
 service cloud.firestore {
   match /databases/{database}/documents {
-    match /{document=**} {
-      allow read, write;
+    match /users/{user} {
+      allow read, write: if user == request.auth.uid
     }
   }
 }
```

[Git commit](https://github.com/JakeHedman/firestore-security-rules-testing/commit/security-rules)

## Firestore project setup

Create a node project in `firestore/` and install test dependencies.

```sh
cd firestore
npm init
npm i jest @firebase/testing firebase-tools
```

Define the test script in `package.json`:

```diff
   "scripts": {
-    "test": "echo \"Error: no test specified\" && exit 1"
+    "test": "firebase emulators:exec --only firestore \"env PATH=$PATH jest --colors\""
   },
```

[Git commit](https://github.com/JakeHedman/firestore-security-rules-testing/commit/firestore-project-setup)

## Writing tests

Create `firestore.test.js` and write your first tests:

```js
const { apps, initializeTestApp, assertFails } = require('@firebase/testing')

const user = initializeTestApp({
  projectId: `testing-${+new Date()}`,
  auth: { uid: 'userid' },
}).firestore()

afterAll(() => {
  return Promise.all(apps().map(app => app.delete()))
})

describe('User', () => {
  it('Should be able to write own user', async () => {
    await user
      .collection('users')
      .doc('userid')
      .set({ name: 'jake' })
  })

  it('Should not be able to write other users', async () => {
    await assertFails(
      user
        .collection('users')
        .doc('otheruser')
        .set({ name: 'jake' })
    )
  })
})
```

Install the firestore emulator and run the tests locally:

```sh
firebase setup:emulators:firestore
npm test
```

[Git commit](https://github.com/JakeHedman/firestore-security-rules-testing/commit/writing-tests)

## CI Config

Create `.circleci/config.yml` in the project root and add some configuration:
```yaml
version: 2
jobs:
  test:
    docker:
      - image: circleci/node:10
    working_directory: ~/project/firestore
    steps:
      - checkout:
          path: ~/project
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "package-lock.json" }}
      - run: npm install
      - save_cache:
          paths:
            - node_modules
          key: v1-dependencies-{{ checksum "package-lock.json" }}
      - run: npx firebase setup:emulators:firestore
      - run: npm test
workflows:
  version: 2
  test:
    jobs:
      - test
```

It's time to `git commit` and `git push` since circle-ci wants your
`.circleci/config.yml` to exist before you set up your project.

[Git commit](https://github.com/JakeHedman/firestore-security-rules-testing/commit/ci-config)

## Firebase auth in CI

Firebase requires you to be logged in to run the firestore simulator (?!), so
you'll need to generate a firebase token and add it to your circle-ci project.

```sh
firebase login:ci
```

Go to https://circleci.com and set up your project, then find `Project
Settings` -> `Environment Variables` -> `Add Variable` and add your firebase
token.

*Name*: `FIREBASE_TOKEN`

*Value*: Your firebase token (e.g. `1/JA003GorjJAKEhhHeyLove-BRJ5oEROn3`)

Note: This token will not be available in pull requests. If you need to test
PRs you'll need to add your `FIREBASE_TOKEN` to your `.circleci/config.yml`
which will obviously a bad idea since it will get scraped and your firebase
account will be used for crypto mining. If you really need to do this you
should probably create a seperate google account for this purpose only (and no
billing!).

## Running the first CI build

Your first circle-ci build should have failed with `Error: Command requires
authentication, please run firebase login`. Click "Rerun workflow" and watch it
succeed now that we have added auth!
