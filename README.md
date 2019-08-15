# About this guide

The purpose of this guide is to explain the process by which I determined and implemented the necessary updates to the Bus Schedule project. My intended audience for this guide is a developer who is responsible for maintaining the Bus Schedule project.

My general approach is to upgrade one major version at a time, ensuring that the application works and looks the same as it did in the previous version. I also ensure that the project's test suites run with the same results as the previous version.

Please note that this guide uses the [semantic versioning syntax](https://docs.npmjs.com/misc/semver) used by the [Node.js package manager (npm)](https://www.npmjs.com/).

You may see some code blocks in this guide with a leading `$` indicating that the following command is one that you should run in a shell your machine. If one or more lines follow that command, they display the output of running that command on my machine. For example:

```sh
$ node --version
v12.6.0
```

Indicates that I ran the command `node --version` and the output of that command was `v12.6.0`. For reference, I am running this on Mac OS X, as you can see below.

As I discuss the process of upgrades, I am editing files and making commits. You may need to review the git history to see the details of the changes I made to successfully update the application and its test suites.

# About the author

I'm Cliff Rodgers. You can this repository at <https://github.com/kliph/upgrading-angular-from-4-to-8>.

# Updating Angular from 4.2.6 to 5.0

Angular ^4.0.0 is [no longer under support](https://angular.io/guide/releases#support-policy-and-schedule). The current stable version at the time of writing this guide is v8.2.0. To check the current version of Angular when you are reading this, follow the [instructions here](https://angular.io/guide/updating#finding-the-current-version-of-angular).

The [Angular documentation recommends incrementing one major version at a time](https://angular.io/guide/releases#supported-update-paths).

I am currently running Angular 4.2.6. I checked this by running `ng version`. You can see the output of that command below. Your machine may have different library versions installed, so make a note of the output because we will be using it shortly.

```sh
$ npm run ng version

> bus-schedule@1.2.0 ng /Users/kliph/projects/bus-schedule
> ng "version"

    _                      _                 ____ _     ___
   / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
  / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
 / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
/_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
               |___/
@angular/cli: 1.2.0
node: 12.6.0
os: darwin x64
@angular/animations: 4.2.6
@angular/common: 4.2.6
@angular/compiler: 4.2.6
@angular/core: 4.2.6
@angular/flex-layout: 2.0.0-beta.8
@angular/forms: 4.2.6
@angular/http: 4.2.6
@angular/material: 2.0.0-beta.6
@angular/platform-browser: 4.2.6
@angular/platform-browser-dynamic: 4.2.6
@angular/router: 4.2.6
@angular/cli: 1.2.0
@angular/compiler-cli: 4.2.6
@angular/language-service: 4.2.6
```

We'll start out by updating from our current version (4.2.6) to the next major version (5.0).

There is a handy [Angular Update Guide application](https://update.angular.io/) for this purpose. You will need to select the Basic option for App Complexity (because this project does not use any of the Medium or Advanced App Complexity features) and the "I use Angular Material" option under Other Dependencies (because this project uses the `@angular/material` library). You can choose your desired Package Manager. I will use `npm` because it is mentioned in this project's `README.md`.

![Angular Update Guide settings](./images/angular-update-guide-settings.png)

You can see the steps we need to do [here](https://update.angular.io/#4.2:5.0).

Please follow the "Before Updating" and "During the Update" steps in the guide, with the exception of the Angular CLI updates (we will address those below), and then refer to the Additional Steps and Notes for updating Angular from 4.2.6 to 5.0 section below before proceeding.

## Additional Steps and Notes for updating Angular from 4.2.6 to 5.0
Please follow these additional steps before moving on to the next major version upgrade to ensure that the application and test suite work.

### Audit warnings
During the process of the update, you may see a warning like

```sh
found 152 vulnerabilities (50 low, 11 moderate, 91 high)
  run `npm audit fix` to fix them, or `npm audit` for details
```

Don't worry about fixing those for now. We will audit and update those libraries as necessary after we update to the latest stable version of Angular.

### Updating the CLI
In order to run the app and make the tests work, I had to upgrade the CLI. Note that the Angular framework libraries and CLI libraries use aligned version numbering systems after Angular v7. We'll use the [CLI version that was stable around the time Angular v5.2.11 was released](https://www.npmjs.com/package/@angular/cli/v/1.6.6).

```sh
$ npm install @angular/cli@1.6.6
$ npm install ajv@^6.9.1 --only=dev
```

We need to add the `ajv` library to fix a warning that the `ajv-keywords` library requires a peer dependency of `ajv` that is not installed. It has been my experience that dealing with these sorts of issues with hoisted peer dependencies becomes a non-issue when using [Yarn](https://yarnpkg.com/en/) instead of `npm`. I see that it was used in this project previously, and I would recommend considering readopting it.

### E2E tests
Updating the CLI fixes issues running the application and tests after updating. However, I see the following error when running `npm run e2e`:

```sh
[15:08:37] E/launcher - Error: TSError: ⨯ Unable to compile TypeScript
Cannot find type definition file for 'jasmine'. (2688)
Cannot find type definition file for 'jasminewd2'. (2688)
Cannot find type definition file for 'node'. (2688)
e2e/app.e2e-spec.ts (1,33): Cannot find module './app.po'. (2307)
e2e/app.e2e-spec.ts (3,1): Cannot find name 'describe'. (2304)
e2e/app.e2e-spec.ts (6,3): Cannot find name 'beforeEach'. (2304)
e2e/app.e2e-spec.ts (10,3): Cannot find name 'it'. (2304)
e2e/app.e2e-spec.ts (12,5): Cannot find name 'expect'. (2304)
```

Please note that I'm showing a snippet of the full error message above.

We will need to upgrade the `ts-node` library to allow the e2e tests to run. Since the latest version of `ts-node`, 8.3.0, is compatible with versions of all of the subsequent versions of the `typescript` library that we will encounter while updating Angular, we'll use the latest `ts-node`:

```sh
npm i ts-node@latest
```

### Legacy HttpModule and Http service
According the Angular Update Guide, we should switch from using the legacy `HttpModule` and the `Http` service to `HttpClientModule` and the `HttpClient` service. This will require updating the code to work with new `HttpClient` API.

The general approach that we will take is to replace:

```ts
import { HttpModule, Http, ... } from '@angular/http'
```

with

```ts
import { HttpClientModule, HttpClient, ... } from '@angular/common/http'
```

Please note the new path `@angular/common/http`.

More specifically I had to refactor the Routes and Vehicle Locations services to work with the new HttpClient API's testing approach. For your reference, I followed the example in the [Angular Fundamentals guide](https://v5.angular.io/guide/http#testing-http-requests).

### Angular Material
We will need to upgrade the [Angular Material library to a version that was stable around the time that Angular v5.2.11 was released](https://www.npmjs.com/package/@angular/material/v/5.2.5). We'll also need to add the `@angular/cdk` dependency.

```sh
$ npm install --save @angular/material@5.2.5 @angular/cdk@5.2.5
```

Some modules' names have changed. In general, modules that had the prefix `Md` now use the prefix `Mat`. For example, we will need to update `MdIconModule` to `MatIconModule`.

Similarly, the naming scheme for Angular Material components has changed. In general, components that were preivously prefixed with `md-` are now prefixed by `mat-`. For example, we will need to update `<md-sidenav>` to `<mat-sidenav>`.

I also found that I needed to update the styling of the `bus-icon-button` to use `display: flex` in order to keep styling consistent with the outdated version of the application.

Next we will need to add the `BrowserAnimationsModule` to `app.module.ts` and ensure that it is included in the imports for the `AppComponent` spec.

Import the `BrowserAnimationsModule` in `app.module.ts`:

```ts
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';
```

And include it in the module's imports:

```ts
@NgModule({
...
  imports: [
    ...
    BrowserAnimationsModule,
    ...
  ],
...
```

These changes will be sufficient to ensure that specs pass and the e2e specs run, however the application will not render. There is a race condition where the Vehicle Location Map component expects to find a div with the id `vehicle-location-map`, but it is not present. We can solve this by moving the method calls in the lifecycle hook used by the Vehicle Location Map component from `ngOnInit` to `ngAfterViewInit`.

We will also need to update the interfaces that the `VehicleLocationMapComponent` class to replace `OnInit` with `AfterViewInit`:

```ts
export class VehicleLocationMapComponent implements OnDestroy, AfterViewInit {
```

And the relevant imports become:

```ts
import { Component, OnDestroy, AfterViewInit } from '@angular/core';
```

I have taken the liberty of adding a test to ensure that `ngAfterViewInit` calls the expected methods for creating the Google Map element and setting up the necessary data subscriptions. You can find the test in `vehicle-location-map.component.spec.ts`.

### Angular Flex-Layout
You can remove the `@angular/flex-layout` library because it is not used in this application.

```sh
$ npm uninstall @angular/flex-layout
```

### Angular Language Service
You can upgrade Angular Language Service by running the following command:

```sh
$ npm i @angular/language-service@5.2.11
```

### Upgrading codelyzer

You can upgrade `codelyzer` by running the following command:

```sh
npm install codelyzer@4.0.2
```

As of [Codelyzer v4.0.0](https://github.com/mgechev/codelyzer/commit/4d495a0d1912b87a4aca340b713a2e3360098afb#diff-4ac32a78649ca5bdd8e0ba38b7006a1e) the `no-access-missing-member`, `templates-use-public`, and `invoke-injectable` linter rules were removed. We can delete them from the `tslint.json` configuration.

# Updating Angular from 5.2.11 to 6.0

Here are the versions of Angular libraries that I have installed.

```sh
$ npm run ng version

> bus-schedule@1.2.0 ng /Users/kliph/projects/bus-schedule
> ng "version"


    _                      _                 ____ _     ___
   / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
  / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
 / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
/_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
               |___/

Angular CLI: 1.6.6
Node: 12.6.0
OS: darwin x64
Angular: 5.2.11
... animations, common, compiler, compiler-cli, core, forms
... http, language-service, platform-browser
... platform-browser-dynamic, platform-server, router

@angular/cdk: 5.2.5
@angular/cli: 1.6.6
@angular/material: 5.2.5
@angular-devkit/build-optimizer: 0.0.42
@angular-devkit/core: 0.0.29
@angular-devkit/schematics: 0.0.52
@ngtools/json-schema: 1.1.0
@ngtools/webpack: 1.9.6
@schematics/angular: 0.1.17
typescript: 2.4.2
webpack: 3.10.0
```

You can see [the steps we need to take to update from 5.2.11 to 6.0 here](https://update.angular.io/#5.2:6.0). Please note that we have already completed the Before Updating step involving switching to `HttpClientModule` and `HttpClient`.

You may notice that the instructions in the "During the Update" step are vague.

**Please do not follow the steps in the Angular Update Guide.** Please instead follow the Additional Steps and Notes for updating Angular from 5.2.11 to 6.0 section before proceeding.

## Additional Steps and Notes for updating Angular from 5.2.11 to 6.0

### Ensure you're running Node 8 or later

Follow the instructions [here](http://www.hostingadvice.com/how-to/update-node-js-latest-version/).

### Updating the CLI
We'll use the [long-term support CLI version for Angular 6](https://www.npmjs.com/package/@angular/cli). At the time of creating this guide, that version is v6.2.9.

```sh
$ npm i @angular/cli@6.2.9
```

Then we'll run the Angular CLI update tool to update the CLI.

```sh
$ npm run ng update @angular/cli
```

### Updating Angular framework packages
We'll start by updating all of the Angular framework packages to ^6.0.0.

```sh
$ npm install @angular/{animations,common,compiler,compiler-cli,core,forms,http,platform-browser,platform-browser-dynamic,platform-server,router,language-service}@^6.0.0 typescript@2.9.2 rxjs@^6.0.0
$ npm install typescript@2.9.2 --save-exact
$ npm install tslint@^5.5
$ npm install @angular/material@^6.0.0 @angular/cdk@^6.0.0
$ npm install -g rxjs-tslint
$ rxjs-5-to-6-migrate -p src/tsconfig.app.json
```

### Fixing rxjs import paths

For now, we will use [`rxjs-compat`](https://github.com/ReactiveX/rxjs/blob/6.x/compat/README.md) to maintain compatibility for some older third-party libraries that we will eventually update and/or replace. But first, in the codebase, we will update the imports from the `rxjs` library for compatability with v6.5.2 in order to maintain compatibility moving forward.

The only file that I found that was missed by the `rxjs-5-to-6-migrate` utility was `vehicle-location-map.component.spec.ts`.

I needed to change the relevant import from:

```ts
import { Subject } from 'rxjs/Subject';
```

to:

```ts
import { Subject } from 'rxjs';
```

Now we can add the `rxjs-compat` dependency:

```sh
$ npm install rxjs-compat@^6 --save
```

### Updating xml2js compatibility

The `xml2js` library that we use to parse XML requires a couple of additional libraries to run in properly in the browser following these updates. Please see [this github comment](https://github.com/Leonidas-from-XIV/node-xml2js/issues/301#issuecomment-479531282) for more information.

You will need to install the `timers-browserify` and `stream-browserify` depedencies. These libraries add `Node.js`-like functionality to the browser.

```sh
$ npm install timers-browserify stream-browserify --save
```

And then you will need to update your `tsconfig.json`. The relevant updates are:

```json
{
  ...
  "compilerOptions": {
    ...
    "paths": {
      "timers": [
        "node_modules/timers-browserify"
      ],
      "stream": [
        "node_modules/stream-browserify"
      ]
    },
    ...
  },
  ...
}
```

I would recommend reviewing the `xml2js` dependency to determine whether it is appropriate for your needs going forward. If there are other alternative libraries for parsing XML that do not require additional dependencies providing `Node.js`-like functionality for browsers, it probably makes sense to migrate away from `xmls2js`.

### Remove deprecated tslint rule
I removed the `typeof-compare` rule because it is redundant, according to its documentation.

It appears that there is [active discussion about whether the `no-use-before-declare` rule should be deprecated or undeprecated here](https://github.com/palantir/tslint/issues/4789). I would recommend following this discussion and determining for yourself whether you should remove this rule.

I was able to demonstrate that the `no-use-before-declare` linter rule catches what would result in a run-time error by adding the following code in a file that the linter checks:

```ts
const consumer = () => {
  console.log(variable);
}

consumer();

const variable = 'foo';
```

Based on this helpful behavior, I decided to keep it for now.

### Map Marker issues

I have observed what appear to be duplicate Map Markers and Map Markers that remain after deselecting all Routes. It is unclear to me whether this is existing behavior in the application or a result of the updates. Refreshing the application seems to restore normal working behavior of showing accurate Map Markers and removing the Map Markers for a given Route upon deselection.

I would recommend investigating these behaviors further in order to determine their root causes. If these behaviors are not intended, I would recommend trying to capture the intended behavior in a regression test that is sensitive to the unintended behavior.

# Angular 2 local storage
You may notice the following warnings when running `npm`:

```sh
npm WARN angular-2-local-storage@1.0.2 requires a peer of @angular/common@^5.2.10 but none is installed. You must install peer dependencies yourself.
npm WARN angular-2-local-storage@1.0.2 requires a peer of @angular/core@^5.2.10 but none is installed. You must install peer dependencies yourself.
npm WARN angular-2-local-storage@1.0.2 requires a peer of @angular/platform-browser@^5.2.10 but none is installed. You must install peer dependencies yourself.
npm WARN angular-2-local-storage@1.0.2 requires a peer of rxjs@^5.5.10 but none is installed. You must install peer dependencies yourself.
```

I would recommend investigating alternative solutions for using local storage on Angular versions after v6. I have observed the application continues to successfully persist Route selection in Local Storage in all of the updates I cover in this guide, but I anticipate that this behavior could change with future updates to the Angular ecosystem, i.e. v9+.

# Updating Angular from 6.1.10 to 7.0

Here are the versions of Angular libraries that I have installed.

```sh
$ npm run ng version

> bus-schedule@1.2.0 ng /Users/kliph/projects/bus-schedule
> ng "version"


     _                      _                 ____ _     ___
    / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
   / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
  / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
 /_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
                |___/


Angular CLI: 6.2.9
Node: 12.6.0
OS: darwin x64
Angular: 6.1.10
... animations, common, compiler, compiler-cli, core, forms
... http, language-service, platform-browser
... platform-browser-dynamic, platform-server, router

Package                           Version
-----------------------------------------------------------
@angular-devkit/architect         0.8.9
@angular-devkit/build-angular     0.8.9
@angular-devkit/build-optimizer   0.8.9
@angular-devkit/build-webpack     0.8.9
@angular-devkit/core              0.8.9
@angular-devkit/schematics        0.8.9
@angular/cdk                      6.4.7
@angular/cli                      6.2.9
@angular/material                 6.4.7
@ngtools/webpack                  6.2.9
@schematics/angular               0.8.9
@schematics/update                0.8.9
rxjs                              6.5.2
typescript                        2.9.2
webpack                           4.16.4
```

You can see [the steps we need to take to update from 6.1.10 to 7.0 here](https://update.angular.io/#6.1:7.0). Please note that we have already completed the Before Updating steps involving switching to `HttpClientModule` and `HttpClient` and removing deprecated RxJS features.

**Please do not follow the steps in the Angular Update Guide.** Please instead follow the Additional Steps and Notes for updating Angular from 6.1.10 to 7.0 section before proceeding.

## Additional Steps and Notes for updating Angular from 6.1.10 to 7.0

### Updating Angular framework packages and peer dependencies
We'll start by updating all of the Angular framework packages to ^7.0.0.

```sh
$ npm install @angular/{animations,common,compiler,compiler-cli,core,forms,http,platform-browser,platform-browser-dynamic,platform-server,router,language-service}@^7.0.0 typescript@^3.1.6
$ npm install @angular/material@^7.0.0 @angular/cdk@^7.0.0
$ npm install @angular/cli@^7.0.0
$ npm install @ngtools/webpack@^7.0.0
$ npm install codelyzer@~4.5.0
$ npm install zone.js@~0.8.26
$ npm i @angular-devkit/build-angular@~0.13.9 --only=dev
```

### Remove unnecessary dependency

We no longer need the `ajv` dependency, so we can remove it from our `package.json`.

# Updating Angular from 7.2.15 to 8.0

Here are the versions of Angular libraries that I have installed.

```sh
$ npm run ng version

> bus-schedule@1.2.0 ng /Users/kliph/projects/bus-schedule
> ng "version"


     _                      _                 ____ _     ___
    / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
   / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
  / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
 /_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
                |___/


Angular CLI: 7.3.9
Node: 12.6.0
OS: darwin x64
Angular: 7.2.15
... animations, common, compiler, compiler-cli, core, forms
... http, language-service, platform-browser
... platform-browser-dynamic, platform-server, router

Package                           Version
-----------------------------------------------------------
@angular-devkit/architect         0.13.9
@angular-devkit/build-angular     0.13.9
@angular-devkit/build-optimizer   0.13.9
@angular-devkit/build-webpack     0.13.9
@angular-devkit/core              7.3.9
@angular-devkit/schematics        7.3.9
@angular/cdk                      7.3.7
@angular/cli                      7.3.9
@angular/material                 7.3.7
@ngtools/webpack                  7.3.9
@schematics/angular               7.3.9
@schematics/update                0.13.9
rxjs                              6.5.2
typescript                        3.1.6
webpack                           4.29.0
```

You can see [the steps we need to take to update from 7.2.15 to 8.0 here](https://update.angular.io/#7.2:8.0). Please note that we have already completed the Before Updating steps involving switching to `HttpClientModule` and `HttpClient` and removing deprecated RxJS features.

**Please do not follow the steps in the Angular Update Guide.** Please instead follow the Additional Steps and Notes for updating Angular from 7.2.15 to 8.0 section before proceeding.


## Additional Steps and Notes for updating Angular from 7.2.15 to 8.0

### Update @ngtools/webpack

We'll update `@ngtools/webpack` to the latest version:

```sh
$ npm i @ngtools/webpack@latest
```

### Update Angular framework and CLI

We can finally use the `ng update` tool:

```sh
$ npm run ng update @angular/cli @angular/core
```

I previously ran into issues using the `--from` and `--to` parameters with the `ng update` tool. I observed that the tool attempted to update to v9 (tagged as `next`) regardless of the parameters I passed as `--from` and `--to`. I did not try to investigate the cause of this issue.

Based on this experience, I strongly recommend closely monitoring [the major version releases](https://blog.angular.io/tagged/release%20notes) in order to update while the tooling can default to the next major version.

### New build targets

As mentioned in the Angular Update Guide, please note that "[t]he CLI's build command now automatically creates a modern ES2015 build with minimal polyfills and a compatible ES5 build for older browsers, and loads the appropriate file based on the browser. You may opt-out of this change by setting your `target` back to `es5` in your `tsconfig.json`. Learn more on [angular.io](https://angular.io/guide/deployment#differential-loading)."

### Updating Angular Material

Update Angular Material and the CDK:

```sh
$ npm i @angular/cdk@8.1.2 @angular/material@8.1.2
```

I am not sure why the update guide and the `ng update` tool recommend switching to deep imports.
I have not observed warnings or linting errors that indicate that this is necessary. The Angular Material [documentation](https://github.com/angular/components/blob/master/CHANGELOG.md#package-structure) points out that deep imports are an anti-pattern.

When I talk about deep imports, I mean for example, the `route-item.component.spec.ts` spec will need to update the `MatCheckboxModule` import from shallow:

```ts
import { MatCheckboxModule } from '@angular/material';
```

to deep:

```ts
import { MatCheckboxModule } from '@angular/material/checkbox';
```

I would recommend against switching to deep imports until it is necessary to maintain compatibility with Angular Material. Maintaining deep imports could require more code changes than maintaining shallow imports.

# You should be updated to the latest stable version at the time of writing this guide

```sh
$ npm run ng version

> bus-schedule@1.2.0 ng /Users/kliph/projects/bus-schedule
> ng "version"


     _                      _                 ____ _     ___
    / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
   / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
  / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
 /_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
                |___/


Angular CLI: 8.2.1
Node: 12.6.0
OS: darwin x64
Angular: 8.2.1
... animations, cli, common, compiler, compiler-cli, core, forms
... language-service, platform-browser, platform-browser-dynamic
... platform-server, router

Package                           Version
-----------------------------------------------------------
@angular-devkit/architect         0.802.1
@angular-devkit/build-angular     0.802.1
@angular-devkit/build-optimizer   0.802.1
@angular-devkit/build-webpack     0.802.1
@angular-devkit/core              8.2.1
@angular-devkit/schematics        8.2.1
@angular/cdk                      8.1.2
@angular/material                 8.1.2
@ngtools/webpack                  8.2.1
@schematics/angular               8.2.1
@schematics/update                0.802.1
rxjs                              6.5.2
typescript                        3.5.3
webpack                           4.38.0
```

# Auditing vulnerable libraries

Please follow the instructions [here](https://docs.npmjs.com/auditing-package-dependencies-for-security-vulnerabilities) to audit and upgrade vulnerable dependencies.

# Pinning dependencies
It is a best practice to [pin dependencies](https://before-you-ship.18f.gov/infrastructure/pinning-dependencies/) before deploying your application. I would recommend that you pin the libraries when deploying this application to production.
