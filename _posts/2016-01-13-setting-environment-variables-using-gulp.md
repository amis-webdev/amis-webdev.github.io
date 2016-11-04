---
title: Setting environment variables using Gulp
author: cindy
---

For the project I am working on we have three different environments, which have their own database and users. We don't want this login credentials to be in our git repository, and we also don't want to manually adjust the credentials every time we do a release to another environment.
Our project is done in *Typescript* using the *AngularJS* framework and the building is done with *Gulp*. But this solution will also fit for plain *JavaScript* projects as long as *Gulp* is used for building the scripts.

One of the solutions of using environment variables is using flags. (e.g. `gulp build –prod and gulp build –local`). But with this option the variables should be set in the Gulp file, and thus in the repo. Also, we don't like the idea of using flags, a build should just be build with one command, no matter the environment. We also need to change the variables for the environment, in every environment if we do a git pull since the variables will be overwritten by the pull.

The solution we came up with makes use of `gulp-replace-task`. This plugin allows us to use placeholders in the source code which we can replace with the actual database connection data when building the project.

We have a configuration file somewhere on disk (`configProperties.json`), which contains the database connection information. This file exists in every environment we have.

##### configProperties.json:
{:label}

```json
{
    "dbConnection": {
        "host": "localhost",
        "port": 3306,
        "user": "user",
        "password": "password",
        "database": "database",
        "connectionLimit": 10
    },
    "server": {
        "port": 8081
    }
}
```

In our `config.ts` file we have the placeholders where the data should be.

##### config.ts:
{:label}

```typescript
//this placeholder will be filled out by gulp build
/**Global setting (environment independent) */
export const server = @@server;

export const dbConnection: IPoolConfig = @@dbConnection;
```

Notice that the name of the placeholder and the name of the objects in the json in the `configProperties.json` are the same. This is because `gulp-replace-task` will map it for us.
Since we use typescript, and made a type for the connection, we can still use the autocompletion in the rest of our source code.
In gulp we made the following method using the `gulp-replace-task`.

```javascript
function replaceEnvironmentVars() {
    // Get the JSON settings.
    var jsonEnvSettings = JSON.parse(fs.readFileSync(path.join(__dirname, environmentSettings), 'utf8'));

    console.log('Set environment variables...',
        jsonEnvSettings);
    // Replace the placeholder @@dbConnection with the dbConnection from the json file.          
    return replace({
        patterns: [
            {
                json: jsonEnvSettings
            }
        ]
    });
};
```

This maps the objects in the json to the placeholders in the code, and replaces them in the pipe.
We can now use this function in our build task.
After this you can do whatever you want, in our case we typescript-compile our source code and write it to the dist directory.

```javascript
gulp.task('build', ['clean'], () => {
    gulp.src(['src/ts/**/*.ts', 'typings/**/*.d.ts'], { base: 'src/ts' })
        .pipe(replaceEnvironmentVars())
        .pipe(tsc(tsProject))
        .pipe(gulp.dest('src/dist'));
});
```

The compiled `config.js` now contains the database connection data which was in the `configProperties.json`.

##### config.js:
{:label}

```javascript
//this placeholder will be filled out by gulp build
/**Global setting (environment independent) */
exports.server = { "port": 8081 };
exports.dbConnection = { "host": "localhost", "port": 3306, "user": "user", "password": "password", "database": "database", "connectionLimit": 10 };
```


All we have to do is set up a `configProperties.json` file in every environment and do a gulp build to make the code use it. If, for example, the password changes in the production environment, we can change the configProperties file on the production environment.

Demo application: [demoApp](https://technology.amis.nl/wp-content/uploads/2016/01/demoApp.zip)

Sources:

- <http://gulpjs.org/>
- [gulp-replace-task](https://www.npmjs.com/package/gulp-replace-task)

{:label: style="margin-bottom: 0;"}
