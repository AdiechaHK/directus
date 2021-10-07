# Custom API Hooks <small></small>

> Custom API Hooks allow running custom logic when a specified event occurs within your project. There are different
> types of events to choose from.

## Extension Entrypoint

The entrypoint of your hook is the `index` file inside the `src/` folder of your extension package. It exports a
register function to register one or more event listeners.

Example of an entrypoint:

```js
export default ({ filter, action }) => {
	filter('items.create', () => {
		console.log('Creating Item!');
	});

	action('items.create', () => {
		console.log('Item created!');
	});
};
```

## Events

### Action

An action event executes after a defined event and receives data related to the event. Use actions hooks when you need
to automate responses to CRUD events on items or server actions.

The action register function receives two parameters:

- The event name
- A callback function that is executed whenever the event fires.

The callback function itself receives two parameters:

- An event-specific meta object
- A context object

The context object has the following properties:

- `database` — The current database transaction
- `schema` — The current API schema in use
- `accountability` — Information about the current user

### Filter

Filter hooks act on the event's payload before the event is fired. They allow you to check, modify, or cancel an event.

It also allows you to cancel an event based on the logic within the hook. The following example shows how you can cancel
an event by throwing a standard Directus exception:

```js
export default ({ filter }, { exceptions }) => {
	const { InvalidPayloadException } = exceptions;

	filter('items.create', async (input) => {
		if (LOGIC_TO_CANCEL_EVENT) {
			throw new InvalidPayloadException(WHAT_IS_WRONG);
		}

		return input;
	});
};
```

The filter register function receives two parameters:

- The event name
- A callback function that is executed whenever the event fires.

The callback function itself receives three parameters:

- The modifiable payload
- An event-specific meta object
- A context object

The context object has the following properties:

- `database` — The current database transaction
- `schema` — The current API schema in use
- `accountability` — Information about the current user

### Init

An init event executes at a defined point within the lifecycle of Directus. Use init event objects to inject logic into
internal services.

The init register function receives two parameters:

- The event name
- A callback function that is executed whenever the event fires.

::: warning Performance

Filters can impact performance when not carefully implemented, as they are executed in a blocking manner.

:::

### Action

- An event-specific meta object

### Schedule

A schedule event executes at certain points in time rather than when Directus performs a specific action. This is
supported through [`node-cron`](https://www.npmjs.com/package/node-cron). To set up a scheduled event, provide a cron
statement as the first parameter to the `schedule()` function. For example `schedule('15 14 1 * *', <...>)` (at 14:15 on
day-of-month 1) or `schedule('5 4 * * sun', <...>)` (at 04:05 on Sunday). See example below:

```js
const axios = require('axios');

module.exports = function registerHook({ schedule }) {
	schedule('*/15 * * * *', async () => {
		await axios.post('http://example.com/webhook', { message: 'Another 15 minutes passed...' });
	});
};
```

## Available Events

### Action Events

| Name                          | Meta                                                |
| ----------------------------- | --------------------------------------------------- |
| `server.start`                | `server`                                            |
| `server.stop`                 | `server`                                            |
| `response`                    | `request`, `response`, `ip`, `duration`, `finished` |
| `auth.login`                  | `payload`, `status`, `user`, `provider`             |
| `files.upload`                | `payload`, `key`, `collection`                      |
| `(<collection>.)items.read`   | `payload`, `query`, `collection`                    |
| `(<collection>.)items.create` | `payload`, `key`, `collection`                      |
| `(<collection>.)items.update` | `payload`, `keys`, `collection`                     |
| `(<collection>.)items.delete` | `payload`, `collection`                             |
| `<system-collection>.create`  | `payload`, `key`, `collection`                      |
| `<system-collection>.update`  | `payload`, `keys`, `collection`                     |
| `<system-collection>.delete`  | `payload`, `collection`                             |

::: tip System Collections

`<system-collection>` should be replaced with one of the system collection names `activity`, `collections`, `fields`,
`folders`, `permissions`, `presets`, `relations`, `revisions`, `roles`, `settings`, `users` or `webhooks`.

:::

::: warning Performance

`read` actions can impact performance when not carefully implemented, as a single request can result in a large amount
of database reads.

:::

### Init

::: tip System Collections

`<system-collection>` should be replaced with one of the system collection names `activity`, `collections`, `fields`,
`folders`, `permissions`, `presets`, `relations`, `revisions`, `roles`, `settings`, `users` or `webhooks`.

:::

### Init Events

| Name                   | Meta      |
| ---------------------- | --------- |
| `cli.before`           | `program` |
| `cli.after`            | `program` |
| `app.before`           | `app`     |
| `app.after`            | `app`     |
| `routes.before`        | `app`     |
| `routes.after`         | `app`     |
| `routes.custom.before` | `app`     |
| `routes.custom.after`  | `app`     |
| `middlewares.before`   | `app`     |
| `middlewares.after`    | `app`     |

## Creating a Hook

### 1. Create a Hook File

Custom hooks are dynamically loaded from within your extensions folder. By default, this directory is located at
`/extensions`, but it can be configured within your project's env file to be located anywhere. The hook-id is the name
of your hook.

#### Default Standalone Hook Location

export default ({ schedule }) => {
	schedule('*/15 * * * *', async () => {
		await axios.post('http://example.com/webhook', { message: 'Another 15 minutes passed...' });
	});
};
```

## Register Function

The register function receives an object containing the type-specific register functions as the first parameter:

Using the example from step 2, `action()` is the hook type and it receives two arguments. `items.create` is the API
event that should trigger the hook. It also receives a callback function that says what the hook should do when the API
event occurs.

```js
const axios = require('axios');

module.exports = function registerHook({ action }) {
	action('items.create', () => {
		axios.post('http://example.com/webhook');
	});
};
```

## Example: Sync with External

```js
const axios = require('axios');

export default ({ filter, action }, { services, exceptions }) => {
	const { MailService } = services;
	const { ServiceUnavailableException, ForbiddenException } = exceptions;

	// Sync with external recipes service, cancel creation on failure
	filter('items.create', async (input, { collection }, { schema }) => {
		if (collection !== 'recipes') return input;

		const mailService = new MailService({ schema });

		try {
			await axios.post('https://example.com/recipes', input);
			await mailService.send({
				to: 'person@example.com',
				template: {
					name: 'item-created',
					data: {
						collection: collection,
					},
				},
			});
		} catch (error) {
			throw new ServiceUnavailableException(error);
		}

		input.syncedWithExample = true;

		return input;
	});
};
```
