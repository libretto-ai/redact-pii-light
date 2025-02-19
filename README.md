# redact-pii-light

> This package is a fork of [redact-pii](https://www.npmjs.com/package/redact-pii) with `@google-cloud/dlp` in peer dependencies. More details can be [found here](https://github.com/solvvy/redact-pii/pull/25).

Remove personally identifiable information from text.

### Prerequesites

This library is primarily written for node.js but it should work in the browser as well.
It is written in TypeScript and compiles to ES2017. The library makes use of `async` functions and hence needs node.js 8.0.0 or higher (or a modern browser). If this is a problem for you please open an issue and we may consider adapting the compiler settings to support older node.js versions.

### Simple example (synchronous API)

```
npm install redact-pii-light
```

```js
const { SyncRedactor } = require('redact-pii-light');
const redactor = new SyncRedactor();
const redactedText = redactor.redact('Hi David Johnson, Please give me a call at 555-555-5555');
// Hi NAME, Please give me a call at PHONE_NUMBER
console.log(redactedText);
```

### Simple example (asynchronous / promise-based API)

```js
const { AsyncRedactor } = require('redact-pii-light');
const redactor = new AsyncRedactor();
redactor.redactAsync('Hi David Johnson, Please give me a call at 555-555-5555').then(redactedText => {
  // Hi NAME, Please give me a call at PHONE_NUMBER
  console.log(redactedText);
});
```

## Supported Features

- sync and async API variants
- ability to customize what to use as replacement value for detected patterns
- built in regex based redaction rules for:
  - credentials
  - creditCardNumber
  - emailAddress
  - ipAddress
  - name
  - password
  - phoneNumber
  - streetAddress
  - username
  - usSocialSecurityNumber
  - zipcode
  - url
  - digits
  - > **NOTE**: the built-in redaction rules are mostly applicable for identifying (US-)english PII.
    > Consider using custom patterns or [Google Cloud DLP](https://cloud.google.com/dlp/) if you have non-english PII to redact.
- ability to add custom redaction regex patterns and complete custom redaction functions (both sync and async)
- ability to use [Google Data Loss Prevention ](https://cloud.google.com/dlp/) as advanced custom redactor

## Advanced usage and features

### Customize replacement values

```js
const { SyncRedactor } = require('redact-pii-light');

// use a single replacement value for all built-in patterns found.
const redactor = new SyncRedactor({ globalReplaceWith: 'TOP_SECRET' });
redactor.redact('Dear David Johnson, I live at 42 Wallaby Way');
// Dear TOP_SECRET, I live at TOP_SECRET

// use a custom replacement value for a specific built-in pattern
const redactor = new SyncRedactor({
  builtInRedactors: {
    names: {
      replaceWith: 'ANONYMOUS_PERSON'
    }
  }
});

redactor.redact('Dear David Johnson');
// Dear ANONYMOUS_PERSON
```

### Add custom patterns or redaction functions

Note that the order of redaction rules matters, therefore you have to decide whether you want your custom redaction rules to run `before` or `after` the built-in ones. Generally it's better to put very specialized patterns or functions `before` the built-in ones and more broad / general ones `after`.

```js
const { SyncRedactor } = require('redact-pii-light');

// add a custom regexp pattern
const redactor = new SyncRedactor({
  customRedactors: {
    before: [
      {
        regexpPattern: /\b(cat|dog|cow)s?\b/gi,
        replaceWith: 'ANIMAL'
      }
    ]
  }
});

redactor.redact('I love cats, dogs, and cows');
// I love ANIMAL, ANIMAL, and ANIMAL

// add a synchronous custom redaction function
const redactor = new SyncRedactor({
  customRedactors: {
    before: [
      {
        redact(textToRedact) {
          return textToRedact.includes('TopSecret')
            ? 'THIS_FILE_IS_SO_TOP_SECRET_WE_HAD_TO_REDACT_EVERYTHING'
            : textToRedact;
        }
      }
    ]
  }
});

redactor.redact('This document is classified as TopSecret.');
// THIS_FILE_IS_SO_TOP_SECRET_WE_HAD_TO_REDACT_EVERYTHING

import { AsyncRedactor } from './src/index';

// add an asynchronous custom redaction function
const redactor = new AsyncRedactor({
  customRedactors: {
    before: [
      {
        redactAsync(textToRedact) {
          return myCustomRESTApiServer.redactCustomWords(textToRedact);
        }
      }
    ]
  }
});
```

### Disable specific built-in redaction rules

```js
const redactor = new SyncRedactor({
  builtInRedactors: {
    names: {
      enabled: false
    },
    emailAddress: {
      enabled: false
    }
  }
});
```

### Use Google Data Loss Prevention

[Google Data Loss Prevention (DLP)](https://cloud.google.com/dlp/) has an extensive rule set to identify and redact PII that goes beyond just simple regex patterns. Consider using DLP in-addition to the built-in patterns of redact-pii-light for high value / sensitive data applications.
Also we strongly advice on using DLP if you have to redact non-english data since redact-pii-light's built-in patterns cover mostly US english patterns only and have no support for non-latin characters, whereas DLP has extensive support for international IDs, Chinese and Korean characters etc..
`redact-pii-light` provides a small wrapper `GoogleDLPRedactor` around DLP that can be used seperately or in conjunction with redact-pii-light's built-in patterns.
Note that Google Cloud DLP already also provides a node.js library (https://www.npmjs.com/package/@google-cloud/dlp) that can be used directly to redact data. You have to decide yourself if you want to use the `GoogleDLPRedactor` wrapper or `@google-cloud/dlp` directly. The main differentiators of using `redact-pii-light` / `GoogleDLPRedactor` are:

- `GoogleDLPRedactor` already instantiates `@google-cloud/dlp` with a bunch of sane defaults and infoTypes
- redact-pii-light has a bunch of built-in patterns which can run in addition to DLP infoTypes
- it is easy to add custom patterns or rules to redact-pii-light
- `GoogleDLPRedactor` uses the `.inspectContent` instead of `.deidentifyContent` method of `@google-cloud/dlp` which has a pricing advantage for large scale redaction scenarios since you will be only charged "Inspection Units" and no additional "Transformation Units" (see https://cloud.google.com/dlp/pricing) . redact-pii-light only uses DLP to `identify` PII but does the replacement `transformation` by itself which saves you some 💰💰💰.

In order to use Google Data Loss Prevention you must install the peer dependency yourself with `npm install --save @google-cloud/dlp` since we are no longer bundling it with this package.

#### Use Google Data Loss Prevention only (this won't make use of redact-pii-light's built-in regex patterns)

1. Prequesites:
   You have to have a Google Cloud Project with DLP enabled and you need a _serviceaccount key json-file_ for a service account with the `serviceusage.services.use` permission or `roles/dlp.user` role. For more detailed steps on how to get a valid service account key follow the steps here: https://github.com/googleapis/nodejs-dlp#before-you-begin

2. Set the environment variable `GOOGLE_APPLICATION_CREDENTIALS` and point it to the serviceaccount key. E.g.:
   `export GOOGLE_APPLICATION_CREDENTIALS=./path/to/my-serviceaccount-key.json`

3. Use redact pii

```js
const { GoogleDLPRedactor } = require('redact-pii-light/lib/custom/GoogleDLPRedactor');

const redactor = new GoogleDLPRedactor();

redactor.redactAsync('I live at 123 Park Ave Apt 123 New York City, NY 10002').then(redactedText => {
  console.log(redactedText);
  // I live at STREET_ADDRESS US_STATE City, LOCATION ZIPCODE'
});
```

#### Use Google DLP AND built-in patterns AND a custom pattern

You can create an `AsyncRedactor` and add a `GoogleDLPRedactor` as custom redactor to the `AsyncRedactor`.
That way you are combining redact-pii-light's built-in patterns with Google DLP. The example below additionally adds a custom regexp pattern.

```js
const { AsyncRedactor } = require('redact-pii-light');
const { GoogleDLPRedactor } = require('redact-pii-light/lib/custom/GoogleDLPRedactor');

const redactor = new AsyncRedactor({
  customRedactors: {
    before: [
      new GoogleDLPRedactor(),
      {
        regexpPattern: /\b(cat|dog|cow)s?\b/gi,
        replaceWith: 'ANIMAL'
      }
    ]
  }
});

redactor.redactAsync('I live at 123 Park Ave Apt 123 New York City, NY 10002 and love cats').then(redactedText => {
  console.log(redactedText);
  // I live at STREET_ADDRESS US_STATE City, LOCATION ZIPCODE and love ANIMAL'
});
```

#### Google DLP content size limit

The Google DLP service has a content size limit of 524288 bytes. If the input is over this limit, the `GoogleDLPRedactor` will
by default automatically split the content into smaller batches and then combine the results together again. If this
behavior is undesired, it can be disabled by setting the `disableAutoBatchWhenContentSizeExceedsLimit` option flag to
true:

```js
new GoogleDLPRedactor({ disableAutoBatchWhenContentSizeExceedsLimit: true });
```

There is no intelligence to try to prevent splitting the batches in the middle of a word. If the batch happens to be
split in the middle of a sensitive word then that word may not be redacted. You can always perform your own intelligent
batching prior if needed.

### Contributing

#### Run tests

You can run the tests via `npm run test`. There are are a bunch of tests which require access to Google's DLP API.
They will only be run if you set the `GOOGLE_APPLICATION_CREDENTIALS` environment variable - otherwise they'll be skipped automatically. You can set it via `GOOGLE_APPLICATION_CREDENTIALS=/path/to/keyfile.json npm test`.

## Credits

[solvvy](https://github.com/solvvy) for [https://github.com/solvvy/redact-pii](https://github.com/solvvy/redact-pii).

[Torgny Bjers](https://github.com/tbjers) for [original PR](https://github.com/solvvy/redact-pii/pull/25).

## License

MIT © [The Half Blood Prince](mailto://thehalfbloodprince.github@gmail.com)
