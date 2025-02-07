---
description: A guide on how you can create a panel to send SMS messages via an external API like Twilio.
contributors: Tim Butterfield, Kevin Lewis
---

# Create An Interactive Panel To Send SMS With Twilio

Panels are used in dashboards as part of the Insights module. As well as read-only data panels, they can be interactive
with form inputs.

![An insights panel showing a form called message customer. The form has a dropdown with 4 items selected, and a text box for a message. The button reads send message.](https://marketing.directus.app/assets/f1c2eafa-507b-46f7-b4f7-b2bf62a73621.png)

## Install Dependencies

This particular panel extension builds off of the
[Twilio Custom Endpoint Extension guide](/guides/extensions/endpoints-api-proxy-twilio). Make sure you have access to
these custom endpoints before starting this guide.

Open a console to your preferred working directory and initialize a new extension, which will create the boilerplate
code for your operation.

```
npx create-directus-extension
```

A list of options will appear (choose panel), and type a name for your extension (for example,
`directus-panel-twilio-sms`). For this guide, select JavaScript.

Now the boilerplate has been created, open the directory in your code editor.

## Specify Configuration

Panels have two parts - the `index.js` configuration file, and the `pane.vue` view. The first part is defining what
information you need to render the panel in the configuration.

Open `index.js` and change the `id`, `name`, `icon`, and `description`.

```js
id: 'panel-twilio-sms',
name: 'Twilio SMS',
icon: 'forum',
description: 'Send a SMS from a panel.',
```

Make sure the `id` is unique between all extensions including ones created by 3rd parties - a good practice is to
include a professional prefix. You can choose an icon from the library [here].

With the information above, the panel will appear in the list like this:

<img alt="Twilio SMS - Send a SMS from a panel. A chat icon is shown in the box." src="https://marketing.directus.app/assets/c2863cc4-2e66-44b0-81bd-e9a3f25d22c1.png" style="padding: 6px 7px 8px 8px;">

The Panel will need some configuration to be able to send messages such as the Twilio account, the sending number, where
to find the contacts and some visual customization. In the `options` sections, add two fields to collect the Twilio
Phone Number and Account SID:

```js
{
	field: 'TWILIO_PHONE_NUMBER',
	name: 'Twilio Phone Number',
	type: 'string',
	meta: {
		interface: 'input',
		width: 'half',
	},
},
{
	field: 'TWILIO_ACCOUNT_SID',
	name: 'Twilio Account SID',
	type: 'string',
	meta: {
		interface: 'input',
		width: 'half',
	},
},
```

To fetch the contacts, add a field for selecting a collection using the system-collection interface and a field for
selecting the phone number field using the system-field interface. These will automatically populate the dropdown with
the values from Directus and form the basis for an API call.

For occasions where the user might want to limit the scope of contacts, add a filter field using the system-filter
interface.

```js
{
	field: 'collection',
	type: 'string',
	name: '$t:collection',
	meta: {
		interface: 'system-collection',
		options: {
			includeSystem: true,
			includeSingleton: false,
		},
		width: 'half',
	},
},
{
	field: 'phone_number_field',
	type: 'string',
	name: 'Phone Number',
	meta: {
		interface: 'system-field',
		options: {
				collectionField: 'collection',
				typeAllowList: ['string','integer'],
		},
		width: 'half',
	},
},
{
	field: 'filter',
	type: 'json',
	name: '$t:filter',
	meta: {
		interface: 'system-filter',
		options: {
			collectionField: 'collection',
			relationalFieldSelectable: false,
		},
	},
},
```

There are many ways to implement this panel so customization is key. Add the following options to allow a fixed 'static'
message, a custom button label, batch recipient list and a custom display template for contacts:

```js
{
	field: 'message',
	type: 'text',
	name: 'Message',
	meta: {
		interface: 'input-multiline',
		width: 'full',
	},
},
{
	field: 'button_label',
	name: 'Button Label',
	type: 'string',
	meta: {
		interface: 'input',
		width: 'half',
	},
},
{
	field: 'batch_send',
	name: 'Send to All',
	type: 'boolean',
	meta: {
		interface: 'boolean',
		width: 'half',
	},
	schema: {
		default_value: false,
	},
},
{
	field: 'displayTemplate',
	name: 'Name in list',
	type: 'string',
	meta: {
		interface: 'system-display-template',
		options: {
			collectionField: 'collection',
			placeholder: '{{ field }}',
		},
		width: 'full',
	},
},
```

After the options section, there is the ability to limit the width and height of the panel. Set these to 12 for the
width and 5 for the height.

It is important to include `skipUndefinedKeys` which is a list of system-display-template fields.

This completes the `index.js` file. The output of the options will look like this:

![A long form showing Twilio credential fields, collection and field selection, a filter, message, button information, an optional Send to All checkbox, and a display template.](https://marketing.directus.app/assets/7aeca419-8a9f-4809-bfcc-889be678f0b7.png)

## Prepare the View

Open the `panel.vue` file and import the following functions at the top of the `<script>`:

```js
import { useApi } from '@directus/extensions-sdk';
import { ref, watch } from 'vue';
```

In the `props`, `showHeader` is one of the built-in properties which you can use to alter your panel if a header is
showing. Remove the `text` property and add all the options that were created in the previous file:

```js
props: {
	showHeader: {
		type: Boolean,
		default: false,
	},
	button_label: {
		type: String,
		default: '',
	},
	collection: {
		type: String,
		default: '',
	},
	phone_number_field: {
		type: String,
		default: '',
	},
	message: {
		type: String,
		default: null,
	},
	filter: {
		type: Object,
		default: () => ({}),
	},
	batch_send: {
		type: Boolean,
		default: false,
	},
	displayTemplate: {
		type: String,
		default: '',
	},
	TWILIO_ACCOUNT_SID: String,
	TWILIO_AUTH_TOKEN: String,
	TWILIO_PHONE_NUMBER: String,
},
```

After the `props`, create a `setup(props)` section and create the variables needed:

```js
setup(props){
	const api = useApi();
	const custom_message = ref('');
	const smsConfirmation = ref(false);
	const recipient = ref('');
	const recipients = ref([]);
	const contacts = ref([]);
	const sms_sent = ref(0);
	const sms_error = ref([]);
	const fields = ref([]);
	const template_fields = ref([]);
	const twilio_sid = props.TWILIO_ACCOUNT_SID;
	const twilio_from = props.TWILIO_PHONE_NUMBER;
},
```

_Note: the `api` is defined from `useApi()` and the Twilio SID and phone number are defined in `props`._

Create a `fetchResults` function to perform the API query. This will use the collection, phone field, filter and any
fields in the display template to construct the query, then use the display template again to format the output inside
the contacts constant.

```js
async function fetchResults(){
	fields.value = [`${props.phone_number_field}`];
	if (props.displayTemplate != null) {
		template_fields.value = props.displayTemplate.match(/(\{\{[\s]*.*?[\s]*\}\})/g);
	}

	if (template_fields.value != null) {
		template_fields.value.forEach(field => {
			field = field.replace('{{ ','').replace(' }}','');
			fields.value.push(field);
		});
	}

	try {
		contacts.value = [];
		const query = await api.get(
			`/items/${props.collection}`,
			{
				params: {
					fields: fields.value,
					limit: -1,
					filter: props.filter,
				},
			}
		);
		const res = query.data.data;
		res.forEach(item => {
			contacts.value.push({
				text: displayOutput(item),
				value: item[props.phone_number_field],
			});
			if (props.batch_send) {
				recipients.value.push(item[props.phone_number_field]);
			}
		});
	} catch (err) {
		console.warn(err);
	}
}

fetchResults();
```

After the function, `fetchResults` is called which will build the contact list when the panel is loaded.

If any of these vital properties are changed, the function will need to update the contact list. Use the following code:

```js
watch(
	[
		() => props.collection,
		() => props.filter,
		() => props.phone_number_field,
		() => props.displayTemplate
	],
	() => {
		fetchResults();
	},
);
```

At this point, return the required variables and functions to the Vue template for use later:

```js
return { contacts, recipient, recipients, custom_message, smsConfirmation, sms_sent, sms_error };
```

In the `fetchResults` function there is a reference to `displayOutput` which needs to be created. This function will use
the `displayTemplate` and replace all the placeholders with their values for the given item.

Split it into 2 functions. The first function will loop through all the placeholders discovered previously using regex
and replace them with the results from the second function `parseValue`.

```js
function displayOutput(item){
	let output = props.displayTemplate;
	if(template_fields.value != null){
		template_fields.value.forEach(field => {
			const clean = field.replace('{{ ','').replace(' }}','');
			output = output.replace(field, parseValue(item, clean));
		});
	}
	return output;
};
```

The `parseValue` function splits the key on the period (`.`) separator, then finds the value for each field from the
supplied item object. The value is returned:

```js
function parseValue(item, key){
	if(key.includes(".")){
		let value = item;
		let fields = key.split('.');
		fields.forEach(f => {
			if(value != null){
				value = value[f];
			}
		});
		return value;
	} else {
		return item[key]
	}
};
```

The outcome of the above functions will change "<span v-pre>{{ name }}</span>, <span v-pre>{{ phone_number }}</span>" to
"Tim, +0123456789".

When the phone number field is specified and batch send is disabled, the user will need a way to select a contact or
contacts. Directus has an interface called `v-select`. When items are selected, write the selection into the
`recipients` variable with the following function:

```js
function updateNumbers(value){
	recipients.value = value;
	return;
};
```

When the SMS is ready to be sent, the recipients and message must be collected and posted to the API. This requires the
Twilio proxy custom endpoint which will relay the request to Twilio and return the response. The custom endpoint is
mapped to `/twilio`.

The responses update the constants `sms_sent` and `sms_error`. This can be used later to render a confirmation to the
user.

```js
function sendSMS(){
	sms_sent.value = 0;
	sms_error.value = [];
	const sms_body = props.message?props.message:custom_message.value;
	const sms_recpients = recipients.value;
	if(recipient.value != ''){
		sms_recpients.push(recipient.value);
	}
	sms_recpients.forEach(sms_to => {
		api.post(
			`/twilio/2010-04-01/Accounts/${twilio_sid}/Messages.json`,
			{
				From: twilio_from,
				Body: sms_body,
				To: sms_to,
			}
		).then((rsp) => {
			if(rsp.data.status == "queued"){
					sms_sent.value += 1;
			} else {
				sms_error.value.push({
					recipient: sms_to,
					error: {
						code: rsp.data.code,
						message: rsp.data.message,
					},
				});
			}
		}).catch((error) => {
			sms_error.value.push({
				recipient: sms_to,
				error: error,
			});
			console.log(error);
		});
	});
	return;
};
```

After a successful send, it’s good practice to clear the form. This function resets the `recipient`, `recipients` (if
needed) and `custom_message` constants back to their initial state:

```js
function SMSReset(){
	if(!props.batch_send){
		recipients.value = [];
	}
	recipient.value = '';
	custom_message.value = '';
	return;
};
```

Update the returned constants and functions with the new ones:

```js
return { contacts, recipient, recipients, custom_message, smsConfirmation, sendSMS, updateNumbers, SMSReset, sms_sent, sms_error };
```

## Build the View

Remove all boilerplate code in the `<template>`, and then add a fallback notice if some essential information is
missing. Start with this:

```html
<template>
	<v-notice type="danger" icon="warning" class="sms-notice" v-if="TWILIO_ACCOUNT_SID === undefined || TWILIO_PHONE_NUMBER === undefined">Twilio API Details Missing</v-notice> // [!code ++]
	<div v-else class="twilio-sms" :class="{ 'has-header': showHeader }"> // [!code ++]
	</div> // [!code ++]
</template>
```

### Recipients

There are 3 ways to receive recipients. If no phone number field is supplied, the panel doesn’t know where to find the
numbers. In this case, an input field is required so the user can type a phone number directly into the panel.

```html
<div v-else class="twilio-sms" :class="{ 'has-header': showHeader }">
	<v-input v-model="recipient" placeholder="+0000000000" v-if="phone_number_field == ''"/> // [!code ++]
</div>
```

If a phone number field is supplied, a multi-select interface is supplied. However, if the user wants the panel to
always send to that list of recipients, `batch_send` can be enabled. With this in mind, the select field is rendered
when `batch_send` is disabled.

```html
<div v-else class="twilio-sms" :class="{ 'has-header': showHeader }">
	<v-input v-model="recipient" placeholder="+0000000000" v-if="phone_number_field == ''"/>
	<v-select // [!code ++]
		v-else-if="!batch_send" // [!code ++]
		multiple // [!code ++]
		:model-value="recipients" // [!code ++]
		:items="contacts" // [!code ++]
		:show-deselect="true" // [!code ++]
		placeholder="Select contacts" // [!code ++]
		:allow-other="true" // [!code ++]
		:close-on-content-click="false" // [!code ++]
		:multiple-preview-threshold="3" // [!code ++]
		:value="recipients" // [!code ++]
		@update:model-value="updateNumbers($event)" // [!code ++]
	></v-select> // [!code ++]
</div>
```

- Setting `allow-other` to `true` will allow the user to include an additional number if needed.
- To only allow a single selection, remove `multiple` from this field.
- The third option is enabling batch send. This doesn’t need any input fields for the user so nothing is rendered.
  Instead, the entire list of recipients is added to the constant when the panel is loaded.

### Message

There are 2 ways for the user to enter a message, at the configuration stage, or on the panel directly.

If a message is supplied in the configuration, that message will be sent whenever the button is pressed. The filters can
be used to control who receives this message to avoid duplication. In this situation, no message field is rendered.

If no message is supplied in the configuration, a plain multi-line input field is rendered in the panel.

```html
<v-textarea class="custom-message" v-model="custom_message" v-if="message == null"></v-textarea>
```

### Send Button

To prevent accidental clicks, it’s a good idea to create a confirmation dialog. Use the following code for the send
button:

```html
<v-dialog v-model="smsConfirmation" @esc="smsConfirmation = false; refresh()">
	<template #activator="{ on }">
		<v-button @click="on" v-if="recipients != undefined && recipients.length > 0 && (message || custom_message != '')">
				{{ button_label }}
		</v-button>
		<v-button v-else secondary disabled>{{ button_label }}</v-button>
	</template>
	<!-- Confirmation goes here -->
</v-dialog>
```

The send button has a fallback which does nothing when the message is missing or there aren't any recipients The dialog
is shown when `smsConfirmation` is true.

Using `v-sheet`, a confirmation box appears in the middle which quotes the message and how many recipients. Below that
are the buttons. Cancel will dismiss the confirmation by setting the `smsConfirmation` constant to `false`, whereas the
Confirm button will run the function `sendSMS`. Lastly the Done button will also dismiss the confirmation but also run
the function `SMSReset` which will empty all the fields.

```html
<v-sheet v-if="recipients != undefined">
	<h2 v-if="sms_sent === 0" class="sms-confirm">Send the following message to {{ recipients.length }} recipients?</h2>
	<blockquote v-if="sms_sent === 0" class="sms-message" v-html="message?message:custom_message"></blockquote>
	<!-- Notices goes here -->
	<div class="sms-actions">
		<v-button v-if="sms_sent === 0" secondary @click="smsConfirmation = false">Cancel</v-button> <v-button v-if="sms_sent === 0" @click="sendSMS()">Confirm</v-button>
		<v-button v-if="sms_sent > 0" @click="smsConfirmation = false; SMSReset()">Done</v-button>
	</div>
</v-sheet>
```

Note, using the `sms_sent` and `sms_error` constants, content can be hidden while the response from the API is shown.

Create some notices above the buttons to show the result from the API. Use the `v-notice` type `danger` if any errors
exist inside `sms_error`. Then use the `v-notice` type `success` when `sms_sent` is greater than 0. It’s important to
show both if an error occurred, so the user knows that some messages have been sent and can view the activity in Twilio
for more information.

```html
<v-notice type="danger" icon="warning" v-if="sms_error.length > 0">There was an issue sending {{ sms_error.length }} message{{ sms_error.length > 1?'s':'' }}.</v-notice>
<v-notice type="success" icon="done" v-if="sms_sent > 0">{{ sms_sent }} message{{ sms_sent > 1?'s':'' }} successfully.</v-notice>
```

### Styling

Add the following CSS:

```css
<style scoped>
.twilio-sms {
	height: 100%;
	display: flex;
	flex-direction: column;
	justify-content: space-between;
	padding: 0 1em 1em;
}

.custom-message {
	flex-grow: 1;
	margin: 1em 0;
	max-height: none;
}

.sms-confirm {
	font-weight: bold;
	font-size: 1.3em;
}

.sms-message {
	padding: var(--input-padding);
	border-radius: var(--border-radius);
	border: var(--border-width) solid var(--border-normal);
	margin: 1em 0;
}

.sms-actions {
	text-align: right;
}

.sms-notice {
	margin: 0 1em;
}
</style>
```

When it’s all together, the panel looks like this:

<img src="https://marketing.directus.app/assets/eedcafab-0397-4956-b5ba-6f6d54473e0f.png" alt="A form with a select contacts dropdown and a text box. A disabled button reads Send Message." style="max-width: 400px; padding: 6px 0 0 8px;"/>

The confirmation panel looks like this:

<img src="https://marketing.directus.app/assets/e219e78c-22ee-48c0-bd14-74bc3e3ddbc5.png" alt="Popup box reads 'Send the following message to 3 recipients: This is awesome. With a cancel and confirm button." style="max-width: 400px;"/>

Both files are now complete. Build the panel with the latest changes.

```
npm run build
```

## Add Panel to Directus

When Directus starts, it will look in the `extensions` directory for any subdirectory starting with
`directus-extension-`, and attempt to load them.

To install an extension, copy the entire directory with all source code, the `package.json` file, and the `dist`
directory into the Directus `extensions` directory. Make sure the directory with your extension has a name that starts
with `directus-extension`. In this case, you may choose to use `directus-extension-panel-twilio-sms`.

If you don’t have the Twilio Endpoint Extension, follow the instructions
[here](/guides/extensions/endpoints-api-proxy-twilio).

Restart Directus to load the extension.

:::info Required files

Only the `package.json` and `dist` directory are required inside of your extension directory. However, adding the source
code has no negative effect.

:::

## Use the Panel

From an Insights dashboard, choose **Twilio SMS** from the list.

Fill in the configuration fields as needed:

- Add your Twilio Phone Number
- Add your Twilio Account SID
- (Optional) Choose a Collection and Phone field or leave blank for Manual entry
- (Optional) Filter the records in the collection
- (Optional) Type a static message to always use or leave blank to write a message each time.
- Type a Button Label
- (Optional) Choose whether or not to always send to the whole contact list
- (Optional) Construct a display template for the recipient selection
- Fill out the Panel Header as normal

Save the panel and dashboard. Add your phone number and compose a message. Click the send button, and then confirm the
send.

## Summary

With this panel, SMS messages become available from the touch of a button anywhere on your dashboards. Combined with the
Twilio custom endpoint extension, you can create more panels for other functions from the Twilio API.

## Complete Code

`index.js`

```js
import PanelComponent from './panel.vue';

export default {
	id: 'panel-twilio-sms',
	name: 'Twilio SMS',
	icon: 'forum',
	description: 'Send a SMS from a panel',
	component: PanelComponent,
	options: [
		{
			field: 'TWILIO_PHONE_NUMBER',
			name: 'Twilio Phone Number',
			type: 'string',
			meta: {
				interface: 'input',
				width: 'half',
			},
		},
		{
			field: 'TWILIO_ACCOUNT_SID',
			name: 'Twilio Account SID',
			type: 'string',
			meta: {
				interface: 'input',
				width: 'half',
			},
		},
		{
			field: 'collection',
			type: 'string',
			name: '$t:collection',
			meta: {
				interface: 'system-collection',
				options: {
					includeSystem: true,
					includeSingleton: false,
				},
				width: 'half',
			},
		},
		{
			field: 'phone_number_field',
			type: 'string',
			name: 'Phone Number',
			meta: {
				interface: 'system-field',
				options: {
					collectionField: 'collection',
					typeAllowList: ['string','integer'],
				},
				width: 'half',
			},
		},
		{
			field: 'filter',
			type: 'json',
			name: '$t:filter',
			meta: {
				interface: 'system-filter',
				options: {
					collectionField: 'collection',
					relationalFieldSelectable: false,
				},
			},
		},
		{
			field: 'message',
			type: 'text',
			name: 'Message',
			meta: {
				interface: 'input-multiline',
				width: 'full',
			},
		},
		{
			field: 'button_label',
			name: 'Button Label',
			type: 'string',
			meta: {
				interface: 'input',
				width: 'half',
			},
		},
		{
			field: 'batch_send',
			name: 'Send to All',
			type: 'boolean',
			meta: {
				interface: 'boolean',
				width: 'half',
			},
			schema: {
				default_value: false,
			},
		},
		{
			field: 'displayTemplate',
			name: 'Name in list',
			type: 'string',
			meta: {
				interface: 'system-display-template',
				options: {
					collectionField: 'collection',
					placeholder: '{{ field }}',
				},
				width: 'full',
			},
		},
	],
	minWidth: 12,
	minHeight: 5,
	skipUndefinedKeys: ['displayTemplate'],
};
```

`panel.vue`

```html
<template>
	<v-notice type="danger" icon="warning" class="sms-notice" v-if="TWILIO_ACCOUNT_SID === undefined || TWILIO_PHONE_NUMBER === undefined">Twilio API Details Missing</v-notice>
	<div v-else class="twilio-sms" :class="{ 'has-header': showHeader }">
		<!-- Content goes here -->
		<v-input v-model="recipient" placeholder="+0000000000" v-if="phone_number_field == ''"/>
		<v-select
			v-else-if="!batch_send"
			multiple
			:model-value="recipients"
			:items="contacts"
			:show-deselect="true"
			placeholder="Select contacts"
			:allow-other="true"
			:close-on-content-click="false"
			:multiple-preview-threshold="3"
			:value="recipients"
			@update:model-value="updateNumbers($event)"
		></v-select>
		<v-textarea class="custom-message" v-model="custom_message" v-if="message == null"></v-textarea>
		<v-dialog v-model="smsConfirmation" @esc="smsConfirmation = false;refresh()">
			<template #activator="{ on }">
				<v-button @click="on" v-if="recipients != undefined && recipients.length > 0 && (message || custom_message != '')">
					{{ button_label }}
				</v-button>
				<v-button v-else secondary disabled>{{ button_label }}</v-button>
			</template>
			<v-sheet v-if="recipients != undefined">
				<h2 v-if="sms_sent === 0" class="sms-confirm">Send the following message to {{ recipients.length }} recipients?</h2>
				<blockquote v-if="sms_sent === 0" class="sms-message" v-html="message?message:custom_message"></blockquote>
				<v-notice type="danger" icon="warning" v-if="sms_error.length > 0">There was an issue sending {{ sms_error.length }} message{{ sms_error.length > 1?'s':'' }}.</v-notice>
				<v-notice type="success" icon="done" v-if="sms_sent > 0">{{ sms_sent }} message{{ sms_sent > 1?'s':'' }} successfully.</v-notice>
				<div class="sms-actions" >
					<v-button v-if="sms_sent === 0" secondary @click="smsConfirmation = false">Cancel</v-button> <v-button v-if="sms_sent === 0" @click="sendSMS()">Confirm</v-button>
					<v-button v-if="sms_sent > 0"  @click="smsConfirmation = false;SMSReset()">Done</v-button>
				</div>
			</v-sheet>
		</v-dialog>
	</div>
</template>

<script>
import { useApi } from '@directus/extensions-sdk';
import { ref, watch } from 'vue';
export default {
	props: {
		showHeader: {
			type: Boolean,
			default: false,
		},
		button_label: {
			type: String,
			default: '',
		},
		collection: {
			type: String,
			default: '',
		},
		phone_number_field: {
			type: String,
			default: '',
		},
		message: {
			type: String,
			default: null,
		},
		filter: {
			type: Object,
			default: () => ({}),
		},
		batch_send: {
			type: Boolean,
			default: false,
		},
		displayTemplate: {
			type: String,
			default: '',
		},
		TWILIO_ACCOUNT_SID: String,
		TWILIO_AUTH_TOKEN: String,
		TWILIO_PHONE_NUMBER: String,
	},
	setup(props){
		const api = useApi();
		const custom_message = ref('');
		const smsConfirmation = ref(false);
		const recipient = ref('');
		const recipients = ref([]);
		const contacts = ref([]);
		const sms_sent = ref(0);
		const sms_error = ref([]);
		const fields = ref([]);
		const template_fields = ref([]);
		const twilio_sid = props.TWILIO_ACCOUNT_SID;
		const twilio_from = props.TWILIO_PHONE_NUMBER;

		async function fetchResults(){
			fields.value = [`${props.phone_number_field}`];
			if(props.displayTemplate != null){
				template_fields.value = props.displayTemplate.match(/(\{\{[\s]*.*?[\s]*\}\})/g);
			}

			if(template_fields.value != null){
				template_fields.value.forEach(field => {
					field = field.replace('{{ ','').replace(' }}','');
					fields.value.push(field);
				});
			}

			try {
				contacts.value = [];
				const query = await api.get(
					`/items/${props.collection}`, {
						params: {
							fields: fields.value,
							limit: -1,
							filter: props.filter,
						},
					}
				);

				var res = query.data.data;

				res.forEach(item => {
					contacts.value.push({
						text: displayOutput(item),
						value: item[props.phone_number_field],
					});

					if(props.batch_send){
						recipients.value.push(item[props.phone_number_field]);
					}
				});

			} catch (err) {
				console.warn(err);
			}
		}

		fetchResults();

		watch(() => [props.collection, props.filter, props.phone_number_field, props.displayTemplate], () => {
			console.log("Setting Changes");
			fetchResults();
		});

		return { contacts, recipient, recipients, custom_message, smsConfirmation, sendSMS, updateNumbers, SMSReset, sms_sent, sms_error };

		function displayOutput(item){
			let output = props.displayTemplate;
			if(template_fields.value != null){
				template_fields.value.forEach(field => {
					var clean = field.replace('{{ ','').replace(' }}','');
					output = output.replace(field, parseValue(item, clean));
				});
			}
			return output;
		}

		function parseValue(item, key){
			if(key.includes(".")){
				let value = item;
				let fields = key.split('.');

				fields.forEach(f => {
					if(value != null){
						value = value[f];
					}
				});

				return value;
			} else {
				return item[key]
			}
		}

		function updateNumbers(value){
			recipients.value = value;
			return;
		}

		function sendSMS(){
			sms_sent.value = 0;
			sms_error.value = [];
			let sms_body = props.message?props.message:custom_message.value;
			let sms_recpients = recipients.value;
			if(recipient.value != ''){
				sms_recpients.push(recipient.value);
			}

			sms_recpients.forEach(sms_to => {
				api.post(`/twilio/2010-04-01/Accounts/${twilio_sid}/Messages.json`, {
					From: twilio_from,
					Body: sms_body,
					To: sms_to,
				}).then((rsp) => {
					if(rsp.data.status == "queued"){
						sms_sent.value += 1;
					} else {
						sms_error.value.push({
							recipient: sms_to,
							error: {
								code: rsp.data.code,
								message: rsp.data.message,
							},
						});
					}
					console.log(rsp.data);
				}).catch((error) => {
					sms_error.value.push({
						recipient: sms_to,
						error: error,
					});
					console.log(error);
				});
			});

			return;
		}

		function SMSReset(){
			if(!props.batch_send){
				recipients.value = [];
			}
			recipient.value = '';
			custom_message.value = '';
			return;
		}
	},
};
</script>

<style scoped>
.twilio-sms {
	height: 100%;
	display: flex;
	flex-direction: column;
	justify-content: space-between;
	padding: 0 1em 1em;
}

.custom-message {
	flex-grow: 1;
	margin: 1em 0;
	max-height: none;
}

.sms-confirm {
	font-weight: bold;
	font-size: 1.3em;
}

.sms-message {
	padding: var(--input-padding);
	border-radius: var(--border-radius);
	border: var(--border-width) solid var(--border-normal);
	margin: 1em 0;
}

.sms-actions {
	text-align: right;
}

.sms-notice {
	margin: 0 1em;
}
</style>
```
