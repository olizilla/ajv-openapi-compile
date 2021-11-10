# ajv-openapi-compile

Given an OpenAPI definition, compile into AJV validation modules to be used in environments (like Cloudflare Workers) where `eval` is not available, or a single JavaScript file is desired.

## Install

The normal way:

```shell
npm install ajv-openapi-compile --save-dev
```

## Build With It

You can use the CLI tool, as part of your build process:

```shell
ajv-openapi-compile --definition=/path/to/openapi.json --output=/path/to/compiled.js
```

You can also use in your build scripts:

```js
import { compile } from 'ajv-openapi-compile'
import { readFile, writeFile } from 'node:fs'
const definition = JSON.parse(await readFile('/path/to/openapi.json', 'utf8'))
const { code } = await compile(definition)
await writeFile('/path/to/compiled.js', code, 'utf8')
```

The output from the `compile` function contains only the `code` property, which has all imports/requires resolved and concatenated into the single string.

## Use Built Code

The compiled code is an ES string, and exports `schema` as a map of schema identifiers to validation functions, as well as two lookup functions:

```js
import { schemas, getSchema, getId } from '/path/to/compiled.js'
```

If you know the fully-resolved schema id, you can access the validation function explicitly:

```js
import { schemas } from '/path/to/compiled.js'
const validate = schemas['#/components/schema/error']
const valid = validate({ code: 404 })
if (!valid) console.log(validate.errors)
```

If you don't know the fully-resolved schema id, you can use the `getId` to get it, or `getSchema` to get the validation function directly. These functions will resolve all `$ref` paths to get to the actual schema id/function.

For example, given a definition like this:

```json
{
	"paths": {
		"/users": {
			"get": {
				"responses": {
					"default": { "$ref": "#/components/responses/error" }
				}
			}
		}
	},
	"components": {
		"responses": {
			"content": {
				"application/json": {
					"schema": { "$ref": "#/components/schemas/error" }
				}
			}
		},
		"schemas": {
			"error": {
				"type": "object",
				"properties": {
					"code": { "type": "number" }
				}
			}
		}
	}
}
```

Using the `getSchema` function to get the schema:

```js
import { getId, getSchema } from '/path/to/compiled'
const path = [ 'paths', '/api/v1/users/{userId}', 'get', 'responses', 'default', 'content', 'application/json', 'schema' ]
// get the schema id:
getId(...path) // => "#/components/schema/error"
// or get the validation function:
const validate = getSchema(...path) // => function
```

**Note:** the functions can take either a string, where each path segment is URI-escaped, or a spread array. For example:

- Valid: `getId('path', 'to', 'foo/bar')`
- Also Valid: `getId('path/to/foo%2Fbar')` (the `/` URI-encodes to `%2F`)
- *Not* Valid: `getId([ 'path', 'to', 'foo/bar' ])`

It is recommended to use the array version if possible, since it won't incur the cost of splitting and URI-unescaping every time something is accessed.

## CLI `ajv-openapi-compile`

The CLI takes the following parameters:

- `---definition, -d` (String) - The path to the definition JSON file.
- `---output, -o` (String) - The path to write the compiled AJV validation code.

For convenience and compatability with other tooling, the `definition` parameter also supports importing JavaScript files, and will follow this algorithm:

1. If the file extension is `.json` read and parse as JSON
2. If the file extension is `.yaml` or `.yml` read and parse as YAML (using `js-yaml` internally)
3. Otherwise, attempt importing `* as schema`
4. If `schema.definition` is set, use that
5. Otherwise, try using `schema`
6. Otherwise, try importing the default as `schema` and try that

To be considered valid, the imported schema definition must have a `paths` object, with at least one "Path Object" defined.

## API: `function(definition: Object) => { code: String }`

The function simply takes a valid OpenAPI 3.x object.

It returns an object with the following properties:

- `code: String` - The compiled code, with all `import` and `require` statements resolved and placed inline.

## License

Published and released under the [Very Open License](http://veryopenlicense.com).

If you need a commercial license, [contact me here](https://davistobias.com/license?software=ajv-openapi-compile).
