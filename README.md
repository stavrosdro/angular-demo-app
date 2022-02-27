# AngularDemoApp

In this article we will create a new Angular project and we will configure eslint, prettier and husky. Linters and git hooks are necessary for both personal and company projects. I will use a fresh VSCode installation without a ton of extensions (only prettier is actually needed).

Most of users have a local installation of Node.js in a Windows machine, so I will follow the exact same approach, but as a Docker enthusiast I will provide the equivalent commands. If you use a [docker development environment](https://medium.com/@stavrosdro/docker-development-environment-angular-full-guide-a38ee34fb651), as I do, you will have the same results.

After the following steps we will be ready to:

- [x] Auto-format the code after every file-save action.

- [x] Identify errors with code linter.

- [x] Execute linter and tests before git commit or git push.

- [x] Enjoy a consistent codebase and reduce diffs & merging conflicts.

*The purpose of this article is to make it easy for the team to clone a project and start working on it, without spending time to configure their local settings. The only required extension is Prettier.*

## Create a new project & install dependencies
```
ng new my-app
cd my-app
npm install
npm install --save-dev --save-exact prettier
touch .prettierrc
ng add @angular-eslint/schematics
npm install eslint-plugin-prettier@latest --save-dev
npm install --save-dev eslint-config-prettier
touch .eslintignore
```

Configuration for the `.editorconfig`

```
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 4 # this is the only change I made, to be aligned with prettier
insert_final_newline = true
trim_trailing_whitespace = true

[*.ts]
quote_type = single

[*.md]
max_line_length = off
trim_trailing_whitespace = false
```

Configuration for the `.prettierrc`
```
{
    "trailingComma": "es5",
    "printWidth": 80,
    "singleQuote": true,
    "useTabs": false,
    "tabWidth": 4,
    "semi": true,
    "bracketSpacing": true
}
```

Configuration for the `.eslintignore`
```
package.json
package-lock.json
dist
e2e/**
karma.conf.js
commitlint.config.js
```

## Configure linter, package.json and workspace settings

Now we have to configure the eslint to use prettier as a formatter and update the VSCode local settings.

Update the scripts in the package.json

`"test": "ng test --watch=false --browsers ChromeHeadless"`

`"lint": "npx eslint \"src/**/*.{js,jsx,ts,tsx,html,css,scss}\" --quiet --fix"`

`"format": "npx prettier \"src/**/*.{js,jsx,ts,tsx,html,css,scss}\" --write"`

Use this eslint configuration. The difference with the default is that we add this line `"plugin:prettier/recommended"` for typescript and html files. Also we created a more specific rule regarding the html components where we want to apply the linter's rules.
```
{
  "root": true,
  "ignorePatterns": [
    "projects/**/*"
  ],
  "overrides": [
    {
      "files": [
        "*.ts"
      ],
      "parserOptions": {
        "project": [
          "tsconfig.json"
        ],
        "createDefaultProgram": true
      },
      "extends": [
        "plugin:@angular-eslint/recommended",
        "plugin:@angular-eslint/template/process-inline-templates",
        "plugin:prettier/recommended"
      ],
      "rules": {
        "@angular-eslint/directive-selector": [
          "error",
          {
            "type": "attribute",
            "prefix": "app",
            "style": "camelCase"
          }
        ],
        "@angular-eslint/component-selector": [
          "error",
          {
            "type": "element",
            "prefix": "app",
            "style": "kebab-case"
          }
        ]
      }
    },
    {
        "files": ["*.html"],
        "extends": ["plugin:@angular-eslint/template/recommended"],
        "rules": {}
    },
    {
      "files": [
        "*.component.html"
      ],
      "extends": [
        "plugin:@angular-eslint/template/recommended",
        "plugin:prettier/recommended"
      ],
      "rules": {}
    }
  ]
}
```

Type `CTRL+SHIFT+P`, search for `workspace settings (JSON)` and use the following
```
{
    "editor.codeActionsOnSave": { "source.fixAll.eslint": true },
    "editor.bracketPairColorization.enabled": true,
    "[html]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    },
    "[scss]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    },
    "[typescript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    }
}
```

Now we can run `npm run format` to format your entire workspace. From now on every time you save a file it will be auto formatted. Also we can run `npm run lint` to check our code for linting errors.
## Install Husky & add pre-push git hook

[Husky](https://typicode.github.io/husky/#/) is a great tool for using Git hooks. I recommend to follow the automatic installation and run `npx husky-init && npm install`. This will also create a pre-commit hook. You can create a pre-push hook by running `npx husky add .husky/pre-push "npm test"`

My pre-commit file
```
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npm run lint
npm test
```

My pre-push file
```
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npm test
```

## Bonus

- You can use `"editor.codeActionsOnSave": { "source.organizeImports": true }` rule in workspace settings to sort imports and remove the unused ones. It's a great rule but remember that there are critical files (e.g. test.ts) which must have a specific order. If you are aware of these files the above rule will be a time-saver.
- You can create more complex rules regarding the imports, such-as keep the external imports on top and separate them with the internals with an empty line. These rules should be defined in the `eslintrc.json` but you need an extension to auto apply them.
- If you create more complex rules which are based on certain extensions you can use `extensions.json` file to notify the team about the recommended extensions. Make sure to check if the extensions you need become deprecated and update your codebase appropriately.
- To save a file without formatting type `CTRL+SHIFT+P` and search for *save file without formatting*.
- You can bypass the git hooks by using the `--no-verify`option.

## Walkthrough for Docker Enthusiasts

For the Docker fellowship I started with this Dockerfile

```
FROM node:14-alpine

RUN npm install -g @angular/cli@13

RUN apk add --no-cache git # we need git to install husky

RUN apk add chromium # we need a browser to run the tests

USER node

WORKDIR /app

EXPOSE 4200 49153

CMD tail -f /dev/null
```

Build the docker image with `docker build -t ng-image .` and create your container with `docker run -it -v $(pwd):/app -p 4200:4200 -p 49153:49153 --name ng-container ng-image sh`

You have to follow the above steps in general. I will spot the differences below:

Create application: `ng new angular-demo-app --skip-git`

Update the scripts in the package.json

`"start": "ng serve --host 0.0.0.0 --poll"`

`"test": "ng test --watch=false"`

`"lint": "npx eslint 'src/**/*.{js,jsx,ts,tsx,html,css,scss}' --quiet --fix"`

`"format": "npx prettier 'src/**/*.{js,jsx,ts,tsx,html,css,scss}' --write"`

In order to run the tests you have to make a couple of changes into the `karma.conf.js` file (I added the 4 commented lines).

```
const process = require('process'); // take the process
process.env.CHROME_BIN = '/usr/bin/chromium-browser'; // set env variable

module.exports = function (config) {
  config.set({
    basePath: '',
    frameworks: ['jasmine', '@angular-devkit/build-angular'],
    plugins: [
      require('karma-jasmine'),
      require('karma-chrome-launcher'),
      require('karma-jasmine-html-reporter'),
      require('karma-coverage'),
      require('@angular-devkit/build-angular/plugins/karma')
    ],
    client: {
      jasmine: {
      },
      clearContext: false
    },
    jasmineHtmlReporter: {
      suppressAll: true
    },
    coverageReporter: {
      dir: require('path').join(__dirname, './coverage/angular-demo-app'),
      subdir: '.',
      reporters: [
        { type: 'html' },
        { type: 'text-summary' }
      ]
    },
    reporters: ['progress', 'kjhtml'],
    port: 9876,
    colors: true,
    logLevel: config.LOG_INFO,
    autoWatch: true,
    browsers: ['ChromeHeadlessNoSandbox'], // change browser
    customLaunchers: { // add browser configuration
        ChromeHeadlessNoSandbox: {
          base: 'ChromeHeadless',
          flags: ['--no-sandbox','--disable-setuid-sandbox']
        }
      },
    singleRun: false,
    restartOnFileChange: true
  });
};
```
Now you have the exact same configuration. The last change is to modify the hooks to use the utility container to execute the hooks. 

Open the `pre-commit` and `pre-push` files.

Replace: `npm run lint` with `docker exec -i ng-container sh -c "cd angular-demo-app && npm run lint"`

Replace: `npm test` with `docker exec -i ng-container sh -c "cd angular-demo-app && npm test"`

Now everything is ready! You can run the tests locally with `docker exec -i ng-container sh -c "cd angular-demo-app && npm test"`. Also with the above modifications in husky hooks you are able to commit and push code as before.

Enjoy!!!
