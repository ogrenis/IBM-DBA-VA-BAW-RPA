# **Lab 2: Integrating Watson Assistant with IBM Business Automation Workflow (BAW)**
In this lab we're going to show how you can extend your Watson Assistant chatbot so that it can call out external services using _**IBM Cloud Functions**_. Our assistant will be triggering a managed workflow running on _**IBM Business Automation Workflow (BAW)**_.

In practical terms, we will be setting up some service calls using IBM Cloud Functions _**actions**_ (to trigger the workflow) and then use them from our Watson Assistant skill and its _dialog nodes_.

## Requirements
- [IBM Cloud account](https://cloud.ibm.com/)
- [Previous lab](../1-Basics)

## Agenda
- Setup _**IBM Cloud Functions**_
- Setup _**Watson Assistant**_ to use _**IBM Cloud Functions**_


## Setup _**IBM Cloud Functions**_
**(1)** We need a mechanism by which we can trigger our managed workflow (running on IBM Business Automation Workflow environment) service and pass the user's inputs information to the workflow API. We'll do this by using _**IBM Cloud Functions**_ running on IBM Cloud.

Based on Apache OpenWhisk, IBM Cloud Functions is a Functions as a Service (FaaS) platform that makes it easy to build and deploy serverless applications. IBM Cloud Functions allow you to write lightweight code that executes application logic in a scalable way. You can then run this code on-demand via requests from applications like our _**Watson Assistant**_ chatbot, or automatically in response to events.

Some examples of how we might use IBM Cloud Functions from _**Watson Assistant**_ include:

- Validating information that you collect from the user.
- Doing calculations or string manipulations on user input that are too complex for supported _SpEL_ expression language methods to handle.
- Interacting with an external web service to get information. For example, you might check on the expected arrival time for a flight from an air traffic service.
- Sending requests to an external application, such as a restaurant reservation site, to complete a simple transaction on the user's behalf.

**(2)** Within your IBM Cloud account, go to _**IBM Cloud Functions**_ by selecting the `burger icon` in the top left-hand corner, then click on `Functions`.

![](./images/open_functions.png)

From there, first _**check the pulldown menu at the top right**_.

**!!Make sure the selected location is the same region where you created your Watson Assistant service!!** So, if you used _**London**_ for your Watson Assistant, you should select _London_ also for your IBM Cloud Functions.

Next, click `Start Creating`, then click the `Action` panel to create an action.

![](./images/functions_start_creating.png)

![](./images/create_action.png)

**(3)** Call your new action `Get Workflow Token`, ensure that `Runtime` of **Node.js 10** is selected and hit `Create`.

![](./images/name_action_get_workflow.png)

You'll then be transported to a code editor. Delete all of the default lines of code within the editor, and replace them with these (copy-paste):

```Javascript
/**
  *
  * main() will be run when you invoke this action
  *
  * @param Cloud Functions actions accept a single parameter, which must be a JSON object.
  *
  * @return The output of this action, which must be a JSON object.
  *
  * The port is unique to every environment
  *
  */

const needle = require('needle');

async function main(params) {

var headers = {
    'Content-Type': 'application/json',
    'Host': 'api.eu-gb.apiconnect.appdomain.cloud'
};

var data = '{"refresh_groups": true,"requested_lifetime": 7200}';

var options = {
    method: 'POST',
    headers: headers,
    body: data
};

// Make sure you replace 'your_external_ip' and 'your_external_port' with the values given by your instructor / what you got from Lab 0.
var URL = 'https://api.eu-gb.apiconnect.appdomain.cloud/jkj-org-dev/sb/awib-workflow/login?ip=your_external_ip&port=your_external_port';

  try {

    let response = await needle(URL,headers,options);
    var token = response.body.csrf_token;
    var call_result = {
        "token": token,
        "status": "Success"
    }
    return {token: token, status: "Success"};

  } catch (err) {
    console.log(err)
    return Promise.reject({
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: { message: err.message, status: "Failed" },
    });
    console.log("error test");
  }
}
exports.main = main;
```

This code will call the managed workflow (running on IBM Business Automation Workflow environment) API and get the authentication token to call BAW actions.

**(4)** You only need to make two small changes to this code.

- Replace `your_external_ip` with the value of your environment ip (DNS) given by your instructor / you got from Lab 0. **Note that it needs to be WITHOUT the quotation marks, e.g. _services-emea.skytap.com_, NOT _"services-emea.skytap.com"_**.
- Replace `your_external_port` with the value of your environment port given by your instructor / you got from Lab 0. **Note that your port number might be 4 or 5 numbers long!**.

Your javascript section for the external ip and port should look something like this:

![](./images/set_ip_and_ports.png)

Now hit `Save`.

![](./images/get-token-code-complete.jpg)

Select `Endpoints` from the sidebar, tick the `Enable as Web Action` box, then click `Save`.

![](./images/web_action_workflow_token.png)

**Copy the Web Action URL and save it in a notepad, you will need it later**. It should look something like this:

https://eu-gb.functions.cloud.ibm.com/api/v1/web/jkj-org_dev/default/Get%20Workflow%20Token

**(5)** Now let's create another action. Click on the Actions menu on the top left side of the screen:

<img src="./images/back_to_actions.png" width="75%">

Then repeat the previous process. Create (click "Create" button) new action called: **Start Address Change Workflow** and paste in the following code:

```Javascript
/**
  *
  * main() will be run when you invoke this action
  *
  * @param Cloud Functions actions accept a single parameter, which must be a JSON object.
  *
  * @return The output of this action, which must be a JSON object.
  *
  */

const needle = require('needle');

async function main(params) {

var headers = {
    'Content-Type': 'application/json',
    'BPMCSRFToken': params.token,
    'Host': 'api.eu-gb.apiconnect.appdomain.cloud'
};


var payload = {
    input:[
      {
        name: 'data',
        data: {
          name: params.name,
          business_id: params.business_id,
          street_address: params.street_address,
          postcode: params.postcode,
          city: params.city,
          email: params.email
        }
      },
    ]
  };

var options = {
    method: 'POST',
    headers: headers,
};

// Make sure you replace 'your_external_ip' and 'your_external_port' with the values given by your instructor / what you got from Lab 0.
var URL = 'https://api.eu-gb.apiconnect.appdomain.cloud/jkj-org-dev/sb/awib-workflow/process?model=Handle%20data%20change&container=AWIBWOR&ip=your_external_ip&port=your_external_port';

  try {
   let response = await needle(URL,JSON.stringify(payload),options);
    var success = response.body;
    console.log(response.body);
    return {success};

  } catch (err) {
    console.log(err)
    return Promise.reject({
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: { message: err.message },
    });
    console.log("error test");
  }
}
exports.main = main;

```

This code will use the authentication token from the previous action and call the BAW with the data we enter in the Watson Assistant chatbot.

**(6)** Again, you only need to make two small changes to this code.

- Replace `your_external_ip` with the value of your environment ip (DNS) given by your instructor / you got from Lab 0. **Note that it needs to be WITHOUT the quotation marks, e.g. _services-emea.skytap.com_, NOT _"services-emea.skytap.com"_**.
- Replace `your_external_port` with the value of your environment port given by your instructor / you got from Lab 0. **Note that your port number might be 4 or 5 numbers long!**.

Now hit `Save`.

Select `Endpoints` from the sidebar, tick the `Enable as Web Action` box, then click `Save`.

Copy the Web Action URL and save it in a notepad, you will need it later. It should look something like this:

https://eu-gb.functions.cloud.ibm.com/api/v1/web/jkj-org_dev/default/Start%20Address%20Change%20Workflow


**(7)** We've now successfully created two IBM Cloud Functions *actions* to "talk" to our IBM Business Automation Workflow API and to launch the *Handle Address Change* workflow. The final thing we need to do here is to make the actions callable from our _**Watson Assistant**_ (or in fact, any other application).

Now let's go and use our functions in _**Watson Assistant**_.

If you are still in Actions, click **Functions** to get to your main Functions page.

<img src="./images/back_to_functions.png" width="75%">

## Setup _**Watson Assistant**_ to use _**IBM Cloud Functions**_
As you've already seen, you need to pass security credentials between services and applications in order to use them. In order to call _**IBM Cloud Functions**_ from within _**Watson Assistant**_ _dialogs_ we need to understand their credentials and encode them correctly.

**(1)** Get your _**IBM Cloud Function**_ **API Key** from the `Namespace Settings` option in the left-hand side menu. There's a copy icon available to copy the key to the clipboard.

![](./images/namespace.png)

**(2)** Next you need to open your _**Watson Assistant**_ _skill_. You might already have it open in your browser tab from Lab 1, but if not, you'll need to open it again. Here's brief instructions to do that:
- Click the navigation (hamburger) menu in the left upper corner. This will open the navigation menu.
- Click `Watson` (the last item in the menu).
- Expand `Watson Services`from the left-hand side menu and click `Existing Services`.
- Find your Watson Assistant Service that you created in Lab 1 and click the name of the service to open it.
- Click `Launch Watson Assistant`.
- Select `Skills` from the left-hand side navigation menu and finally click your `B2B Bank Bot` to open your skill in the editor.

Move to `Dialog`of your skill and you will see a node called `Welcome`. Click it to open its configuration.

Under the 'Then set context' there is a variable called `$private` with the value below:
```Javascript
{"myCredentials":{"api_key":"<your-ibm-cloud-functions-api-key>"}}

```

Replace `<your-ibm-cloud-functions-api-key>` with the key you copied earlier in step **(1)**.

![](./images/welcome_node_api_key_new.png)

Your value for the $private variable should look something like:
**{"myCredentials":{"api_key":"16fcbbb5-8124-435f-8326-38a213704870:i3ejktRgIPu2svXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"}}**

**NOTE!!** Make sure to remove also the "<" and ">" signs from the template. Your API key should be just surrounded by quotation marks.

Now regardless of integration type, your chatbot will always start correctly, and it will define the credentials required to call any of our _**IBM Cloud Functions action**_.

**(3)** Find the node called _**Get Workflow Token**_ and click to open its configuration.

![](./images/set_assistant_get_workflow_token_new2.PNG)

Click the `gear symbol` from the first "Assistant responds" row to open the response editor.

The **only** thing you will need to replace here is `<your-get-workflow-token-endpoint>` with the endpoint information you copied for your Get Workflow Token action. If you didn't save the endpoint in you can get the name of your _**endpoint**_ by going back to your _**IBM Cloud Function**_ in IBM Cloud, clicking `Endpoints` from the sidebar (if you're not already on that screen), then copying everything in the **Web Action URL** _after_ _**.../web/**_.

**NOTE!** *****IMPORTANT!*****
Just use the part of the URL **AFTER** the **.../web/** !!!

It should look something like:

`name.lastname_dev/default/Get%20Workflow%20Token.json`

**NOTE2!** *****IMPORTANT!*****
If your endpoint URL is missing `.json` from the end, **you need to add it!!** Also make sure to remove the "<" and ">" signs from the template. Use just the quotation marks to surround your URL as shown in the picture below.

After copying your end point URL to the configuration it should look similar to this:

![](./images/set_assistant_config_2.png)

**(4)** Find the node called _**Thank you for information**_ and click to open its configuration.

![](./images/thank_you.png)

Click "the 3 dots menu" to open JSON editor and replace `<your-start-address-change-workflow-endpoint>` with the endpoint information you copied for your Start Address Change Workflow action. This is where we trigger the managed workflow that will later start the RPA process. It should look something like:

`name.lastname_dev/default/Start%20Address%20Change%20Workflow.json`


**NOTE!** *****IMPORTANT!*****
Just use the part **AFTER** the **.../web/** !!!

**NOTE2!** *****IMPORTANT!*****
If your endpoint URL is missing `.json` from the end, **you need to add it!!** Also make sure to remove the "<" and ">" signs from the template. Use just the quotation marks to surround your URL as shown in the picture below.

![](./images/assistant_conf_2_done_new3.png)

Finally click "the 3 dots menu" again to close the JSON editor and to save your changes.

Fantastic! Continue with lab 3 to get started with the IBM Business Automation Workflow (BAW) and the Robotic Process Automation (RPA) part. Once you have the next labs running you will be able to call your managed workflow from your Watson Assistant chatbot!


## Summary
You've reached the end of this lab! By completing it you've learned how to further enhance your chatbot by calling external services using _**IBM Cloud Functions**_.

Once you have completed the next two labs you will be able to connect with the BAW using your chatbot.
The chatbot will gather the information needed (Company name, company business id, street address, city, postcode) and send them to the BAW to be checked and handled.

This is what the conversation will look like:

<img src="./images/chat1_new.png" width="50%">

<img src="./images/chat2_new.png" width="50%">

<img src="./images/chat3_new.png" width="50%">


If you see the error at the end, it's due to the fact you do not have your workflow environment yet running that your chatbot tries to call.

**NOTE!** If you have already started your BAW in your virtual environment, you might not see the error. Nevertheless, you are done with this lab.

[CONTINUE TO THE NEXT LAB](../3-BAW)
