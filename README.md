# 🎒 Code Migrate
A framework for declaratively writing codebase migrations on JavaScript/NodeJS based projects.

<p align="center">
  <img src="https://user-images.githubusercontent.com/11733036/99916433-de780180-2d12-11eb-8d5a-e0cf77c2dafc.gif" alt="code-migrate"/>
</p>

## Why
Writing an automatic migration script usually takes time. Besides implementing the transformations to the code/configuration, you also need to handle other concerns like publishing a CLI application, generating a report, handling errors during the migration process, writing tests, and more.

Providing a polished experience usually results in a lot of work which we can not always justify. In some cases, maintainers resort to either API stagnation or leaving the heavy lifting to their users. If you're maintaining a library or a toolkit, you'd want your users to upgrade with minimal effort. And you want to write the migration script in a minimal amount of time.

## Features

* Declarative way to define your migration tasks, leaving you with focusing only on the transformation logic.

* The migration is separated into two parts, the first one is processing all of the tasks and the second is writing them to the file-system. This ensures that in case of an error, nothing will be written to the file-system. It also lets the user approve the migration via a prompt from the CLI.

* Even though nothing is written while processing the tasks, all file system operations are written to a virtual file system which makes sure that tasks that depend on each other will work as expected. For example, if you change a file on the first task, the second task will see its updated version.

* Code Migrate creates a beautiful report of the changes sorted by tasks.

* There is a testkit that helps with the process of writing the migration. You can define `__before__` and `__after__` directories and use TDD to implement the migration with fewer mistakes and with a quick feedback loop.

## CLI
```
  Usage
    $ npx code-migrate <path/to/migration.ts>

  Options
    --version, -v   Version number
    --help, -h      Displays this message
    --dry, -d       Dry-run mode, does not modify files
    --yes, -y       Skip all confirmation prompts
    --cwd           Runs the migration on this directory [defaults to process.cwd()]
```

## Node API
### runMigration
Run a migration programatically
```ts
type RunMigration = ({
  cwd: string;
  migrationFilePath: string;
  dry: boolean;
  yes: boolean;
}) => Promise<void>;
```

## Writing a migration file

```ts
import { migrate } from 'code-migrate';

migrate(
  'automatic migration for my library',
  ({ transform, rename, create, remove }) => {
    transform(
      'add the build directory to .gitignore',
      '.gitignore',
      ({ source }) => source.trim() + '\nbuild\n'
    );

    transform(
      'remove "use-strict"; from all .js files',
      '**/*.js',
      ({ source }) => {
        return source.replace(/("|')use-strict("|');?/, '');
      }
    );

    remove('remove babel config', 'babel.config.js');

    rename('rename the main config to cool config', 'main-config.json', () => {
      return 'cool-config.json';
    });

    create('create an .env file', () => {
      return {
        fileName: '.env',
        source: 'HELLO=WORLD\n',
      };
    });
  }
);
```

## Use with global migrate function
The code-migrate runner does not require that you'll import the `migrate` function, it will also work on the global:

```ts
migrate('example migration', ({ create }) => {
  create('add foo.json file', () => {
    return { fileName: 'foo.json', source: JSON.stringify({ foo: 'bar' }) };
  });
});
```

For TypeScript and autocompletion add this line to any `d.ts` file required in your `tsconfig.json` `files`/`include` arrays.

```ts
declare let migrate: import('code-migrate').Migrate;
```

### migrate
Similar to the way test runners work, Code Migrate will expose a global migrate function, you can also import it from `code-migrate` module, which will only work in the context of the runner. Use it to define your migration tasks.

```ts
type Migrate = (
  title: string,
  fn: (
    tasks: {
      transform: Transform,
      rename: Rename,
      remove: Remove,
      create: Create
    },
    options: { cwd: string, fs: VirtualFileSystem }
  ) => void
) => void;

```
### tasks
migration is defined by a series of tasks from the following list

#### transform
Change the source code of a file or multiple files

```ts
type Transform = (
  title: string,
  pattern: Pattern,
  transformFn: TransformFn
) => void;

type TransformFn = ({
  fileName: string;
  source: string;
}) => string | SerializeableJsonObject;
```

#### rename
Change the name of a file/directory or multiple files/directories

```ts
type Rename = (
  title: string,
  pattern: Pattern,
  renameFn: RenameFn
) => void;

type RenameFn = ({ fileName: string }) => string;
```
#### remove
Delete a file/directory or multiple files/directories
```ts
type Remove = (title: string, pattern: Pattern) => void;

```
#### create
Create a new file/directory or multiple files/directories
```ts

type Create = (
  title: string,
  patternOrCreateFn: EmptyCreateFn | Pattern,
  createFn?: CreateFn
) => void;
```

### options
### fs
Virtual file System which implements a subset of the `fs` module API. You can use it to perform custom file system operations that will be part of the migration process. They will only be written at the end of the migration and will relay on former tasks.

### cwd
The working directory in which the migration currently runs.

## Testing
Code Migrate comes with a testkit that lets you write tests for your migration. Use jest or any other test runner to run your suite:

You'll need to create fixtures of the `__before__` and the `__after__` states, the testkit expects those directories and knows how to accept a migration file and a fixtures directory as parameters.

### Example
Let's consider the following file structure:
```
.
├── __after__
│       └── bar.json
├── __before__
│       └── foo.json
├── migration.test.ts
└── migration.ts
```

And the following migration file:
```ts
// migration.ts

migrate('my migration', ({ transform }) => {
  rename(
      'replace foo with bar in all json files',
      '**/*.json',
      ({ filename }) => filename.replace('foo', 'bar');
    );
});
```

The test initializes the testkit which accepts an optional path to the migration file:

```ts
// migration.test.ts

import { createTestkit } from 'code-migrate/testing';
import path from 'path';

const testkit = createTestkit({
  migrationFile: path.join(__dirname, 'migration.ts'),
});

it('should rename foo.json to bar.json', () => {
  testkit.run({ fixtures: __dirname });
});
```

## Future plans and ideas
* `exec` task (run custom commands, like lint --fix)
* `warn`/`check` task that logs a warning to the console
* code-migrate-tools (support AST related operations)
* Consider adding a `read` task for case the user only wants to retrieve information
* Add a way to create scopes for a single logical task

