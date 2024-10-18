# google-spreadsheet

> A fork of The most popular [Google Sheets API](https://developers.google.com/sheets/api/guides/concepts) wrapper for javascript / typescript

## Differences from the original
- DeveloperMetadata is supported
- Retries are supported when rate limited
- Safer handling of numeric headers
- Support for providing valueRenderOption when loading rows & headers
- Support for reading cell & row metadata

[![NPM version](https://img.shields.io/npm/v/google-spreadsheet)](https://www.npmjs.com/package/google-spreadsheet)
[![CI status](https://github.com/theoephraim/node-google-spreadsheet/actions/workflows/ci.yml/badge.svg)](https://github.com/theoephraim/node-google-spreadsheet/actions/workflows/ci.yml)
[![Known Vulnerabilities](https://snyk.io/test/github/theoephraim/node-google-spreadsheet/badge.svg?targetFile=package.json)](https://snyk.io/test/github/theoephraim/node-google-spreadsheet?targetFile=package.json)
[![NPM](https://img.shields.io/npm/dw/google-spreadsheet)](https://www.npmtrends.com/google-spreadsheet)

- multiple auth options (via [google-auth-library](https://www.npmjs.com/package/google-auth-library)) - service account, OAuth, API key, ADC, etc
- cell-based API - read, write, bulk-updates, formatting
- row-based API - read, update, delete (based on the old v3 row-based calls)
- managing worksheets - add, remove, resize, update properties (ex: title), duplicate to same or other document
- managing docs - create new doc, delete doc, basic sharing/permissions
- export - download sheet/docs in various formats

**Docs site -**
Full docs available at [https://theoephraim.github.io/node-google-spreadsheet](https://theoephraim.github.io/node-google-spreadsheet)

---

> 🌈 **Installation** - `pnpm i google-spreadsheet`<br/>(or `npm i google-spreadsheet --save` or `yarn add google-spreadsheet`)

## Examples

_The following examples are meant to give you an idea of just some of the things you can do_

> **IMPORTANT NOTE** - To keep the examples concise, I'm calling await [at the top level](https://v8.dev/features/top-level-await) which is not allowed in some older versions of node. If you need to call await in a script at the root level and your environment does not support it, you must instead wrap it in an async function like so:

```javascript
(async function () {
  await someAsyncFunction();
})();
```

### The Basics

```js
import { GoogleSpreadsheet } from 'google-spreadsheet';
import { JWT } from 'google-auth-library';

// Initialize auth - see https://theoephraim.github.io/node-google-spreadsheet/#/guides/authentication
const serviceAccountAuth = new JWT({
  // env var values here are copied from service account credentials generated by google
  // see "Authentication" section in docs for more info
  email: process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL,
  key: process.env.GOOGLE_PRIVATE_KEY,
  scopes: ['https://www.googleapis.com/auth/spreadsheets'],
});

const doc = new GoogleSpreadsheet('<the sheet ID from the url>', serviceAccountAuth);

await doc.loadInfo(); // loads document properties and worksheets
console.log(doc.title);
await doc.updateProperties({ title: 'renamed doc' });

const sheet = doc.sheetsByIndex[0]; // or use `doc.sheetsById[id]` or `doc.sheetsByTitle[title]`
console.log(sheet.title);
console.log(sheet.rowCount);

// adding / removing sheets
const newSheet = await doc.addSheet({ title: 'another sheet' });
await newSheet.delete();
```

More info:

- [GoogleSpreadsheet](https://theoephraim.github.io/node-google-spreadsheet/#/classes/google-spreadsheet)
- [GoogleSpreadsheetWorksheet](https://theoephraim.github.io/node-google-spreadsheet/#/classes/google-spreadsheet-worksheet)
- [Authentication](https://theoephraim.github.io/node-google-spreadsheet/#/guides/authentication)

### Working with rows

```js
// if creating a new sheet, you can set the header row
const sheet = await doc.addSheet({ headerValues: ['name', 'email'] });

// append rows
const larryRow = await sheet.addRow({ name: 'Larry Page', email: 'larry@google.com' });
const moreRows = await sheet.addRows([
  { name: 'Sergey Brin', email: 'sergey@google.com' },
  { name: 'Eric Schmidt', email: 'eric@google.com' },
]);

// read rows
const rows = await sheet.getRows(); // can pass in { limit, offset }

// read/write row values
console.log(rows[0].get('name')); // 'Larry Page'
rows[1].set('email', 'sergey@abc.xyz'); // update a value
rows[2].assign({ name: 'Sundar Pichai', email: 'sundar@google.com' }); // set multiple values
await rows[2].save(); // save updates on a row
await rows[2].delete(); // delete a row
```

Row methods support explicit TypeScript types for shape of the data

```typescript
type UsersRowData = {
  name: string;
  email: string;
  type?: 'admin' | 'user';
};
const userRows = await sheet.getRows<UsersRowData>();

userRows[0].get('name'); // <- TS is happy, knows it will be a string
userRows[0].get('badColumn'); // <- will throw a type error
```

More info:

- [GoogleSpreadsheetWorksheet > Working With Rows](https://theoephraim.github.io/node-google-spreadsheet/#/classes/google-spreadsheet-worksheet#working-with-rows)
- [GoogleSpreadsheetRow](https://theoephraim.github.io/node-google-spreadsheet/#/classes/google-spreadsheet-row)

### Working with cells

```js
await sheet.loadCells('A1:E10'); // loads range of cells into local cache - DOES NOT RETURN THE CELLS
console.log(sheet.cellStats); // total cells, loaded, how many non-empty
const a1 = sheet.getCell(0, 0); // access cells using a zero-based index
const c6 = sheet.getCellByA1('C6'); // or A1 style notation
// access everything about the cell
console.log(a1.value);
console.log(a1.formula);
console.log(a1.formattedValue);
// update the cell contents and formatting
a1.value = 123.456;
c6.formula = '=A1';
a1.textFormat = { bold: true };
c6.note = 'This is a note!';
await sheet.saveUpdatedCells(); // save all updates in one call
```

More info:

- [GoogleSpreadsheetWorksheet > Working With Cells](https://theoephraim.github.io/node-google-spreadsheet/#/classes/google-spreadsheet-worksheet#working-with-cells)
- [GoogleSpreadsheetCell](https://theoephraim.github.io/node-google-spreadsheet/#/classes/google-spreadsheet-cell)

### Managing docs and sharing

```js
const auth = new JWT({
  email: process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL,
  key: process.env.GOOGLE_PRIVATE_KEY,
  scopes: [
    'https://www.googleapis.com/auth/spreadsheets',
    // note that sharing-related calls require the google drive scope
    'https://www.googleapis.com/auth/drive.file',
  ],
});

// create a new doc
const newDoc = await GoogleSpreadsheet.createNewSpreadsheetDocument(auth, { title: 'new fancy doc' });

// share with specific users, domains, or make public
await newDoc.share('someone.else@example.com');
await newDoc.share('mycorp.com');
await newDoc.setPublicAccessLevel('reader');

// delete doc
await newDoc.delete();
```

## Why?

> **This module provides an intuitive wrapper around Google's API to simplify common interactions**

While Google's v4 sheets API is much easier to use than v3 was, the official [googleapis npm module](https://www.npmjs.com/package/googleapis) is a giant autogenerated meta-tool that handles _every Google product_. The module and the API itself are awkward and the docs are pretty terrible, at least to get started.

**In what situation should you use Google's API directly?**<br>
This module makes trade-offs for simplicity of the interface.
Google's API provides a mechanism to make many requests in parallel, so if speed and efficiency are extremely important to your use case, you may want to use their API directly. There are also many lesser-used features of their API that are not implemented here yet.

## Support & Contributions

This module was written and is actively maintained by [Theo Ephraim](https://theoephraim.com).

**Are you actively using this module for a commercial project? Want to help support it?**<br>
[Buy Theo a beer](https://paypal.me/theoephraim)

### Sponsors

None yet - get in touch!

### Contributing

Contributions are welcome, but please follow the existing conventions, use the linter, add relevant tests, and add relevant documentation.

The docs site is generated using [docsify](https://docsify.js.org). To preview and run locally so you can make edits, run `npm run docs:preview` and head to http://localhost:3000
The content lives in markdown files in the docs folder.

## License

This is free and unencumbered public domain software. For more info, see https://unlicense.org.
