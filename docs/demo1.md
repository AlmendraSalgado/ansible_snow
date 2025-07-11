# Self-Service VM Provisioning with ServiceNow & Ansible EDA

## Prerequisites

1. **Ansible Automation Platform** deployed with both Controller and EDA components.
2. **An active ServiceNow instance** with administrative access.
3. **Authentication token** for secure integration:
    - A token or API key for authenticating requests from ServiceNow to Ansible EDA (configure this in EDA and use it in your ServiceNow REST message).

## Configuration in Ansible Automation Platform

### Section 1: Event-Driven Ansible

In this section, all the configuration steps for the EDA component will be explained.

#### 1. Create Credentials in Ansible EDA

You will need two credentials in Ansible EDA:

1. **EDA Webhook Token (for ServiceNow to EDA authentication):**
   - Go to **Automation Decisions > Infrastructure > Credentials**.
   - Click **Add** to create a new credential.
   - Select **Token Event Stream** as the credential type.
   - Enter a name and paste the token value (you can generate a random token at this step, but make sure to save it for use in the [REST message configuration in ServiceNow](#3-create-the-rest-message)).
   - Save the credential.

2. **Ansible Automation Platform Credential (for EDA to Controller authentication):**
   - Go to **Automation Decisions > Infrastructure > Credentials**.
   - Click **Add** to create a new credential.
   - Select **Red Hat Ansible Automation Platform** as the credential type.
   - Enter a name.
   - Enter the following authentication credentials:
     - **username**: `<AAP username>`,
     - **password**: `<AAP password>`, or
     - **token**: `<AAP token>`
     - **host**: `<AAP URL>/api/controller/`
   - Save the credential.

#### 2. Create an Event Stream in Ansible Automation Platform (AAP)

To receive events from ServiceNow in Ansible EDA, you need to create an Event Stream:

1. Navigate to **Automation Decisions > Event Streams** in the Ansible Automation Platform web UI.
2. Click **Create event stream** to create a new event stream.
3. Enter a **Name** and (optionally) a **Description** for the event stream.
4. Select **ServiceNow Event Stream** as the event stream type.
5. Copy the **Webhook URL** that is generated—this is the endpoint you will use in your [REST message configuration in ServiceNow](#3-create-the-rest-message).
6. Configure authentication by selecting the credential you created for webhook authentication.
7. Save the event stream.

#### 3. Create and sync Projects in EDA

1. Navigate to **Automation Decisions > Projects** in the Ansible Automation Platform web UI.
2. Click **Create project** to add a new project.
3. Enter a **Name** and (optionally) a **Description** for the project.
4. Set the **Source Control URL** to the link of your Git repository.
5. Click the **Create project** button to save.
6. Wait for the project to sync. The status will update to show success or failure.

#### 4. Activate the Rulebook Activation

To enable automated event processing, you need to activate your rulebook in Ansible EDA:

1. Navigate to **Automation Decisions > Rulebook Activations** in the Ansible Automation Platform web UI.
2. Click **Create rulebook activation** to add a new rulebook activation.
3. Enter a **Name** for the activation, select the project where your rulebook is stored, and choose the rulebook you want to activate.
4. Select the appropriate **Event Stream** you created earlier.
5. Add the credential you previously created to authenticate against the controller.
6. Ensure **Rulebook activate enabled?** is checked.
7. Click **Create rulebook activation** to save and start the rulebook activation.

### Controller Side

On the Controller side of Ansible Automation Platform, you need to prepare the resources that your EDA rulebook will use to automate provisioning.

## Configuration in ServiceNow

### Prerequisites

- **Admin credentials** for ServiceNow.
- **A dedicated ServiceNow user** for Ansible automation. This user will be assigned to resources created or updated by Ansible. If you already have such a user, you can skip this step.

### 1. Create a User in ServiceNow

This user account will be used by Ansible Automation Platform to interact with ServiceNow and will be assigned as the owner or updater of records created during the provisioning process.

1. Navigate to **User Administration > Users** in ServiceNow.
2. Click the purple **New** button to create a new user.
3. Enter the required information, such as **User ID** and **Email**.
4. Click **Submit** to save the new user.

![ServiceNow User](images/image1.png)

### 2. Create the Service Catalog Item

This catalog item will allow users to request a new virtual machine, capturing all necessary details for automated provisioning.

1. Navigate to **Service Catalog > Catalog Builder** in ServiceNow.
2. Click **Create a new catalog item**.
3. Enter a descriptive **Name** and **Description** for the catalog item (e.g., "Provision New Virtual Machine").
4. Select the appropriate **Catalog** and **Category** to help users find the item easily.
5. Review and **Publish** the catalog item to make it available to users.

![ServiceNow Service Catalog 1](images/image2.png)
![ServiceNow Service Catalog 2](images/image3.png)
![ServiceNow Service Catalog 3](images/image4.png)

### 3. Create the REST Message

The REST message will be used to send events from ServiceNow to Ansible EDA, enabling automated processing of catalog requests.

1. Navigate to **System Web Services > Outbound > REST Message** in ServiceNow.
2. Click **New** to create a new REST message.
3. Enter a **Name** for the REST message (e.g., "Send Event to EDA").
4. In the **Endpoint** field, enter the URL of your Ansible EDA webhook receiver (e.g., `https://<eda-server>/endpoint`). This URL will be provided when you configure the event stream in Ansible Automation Platform (AAP).  
   For instructions on how to obtain this URL, see the [Configuring the Event Stream in AAP](#configuring-the-event-stream-in-aap) section.
5. (Optional) Add a **Description** to clarify the purpose of this REST message.
6. In the **HTTP Request** tab of your REST message method, configure the following:
    - **Authorization**:  
      If your Ansible EDA endpoint requires authentication, select the appropriate authorization type (Bearer) and provide the necessary credentials ([the token created as Credential in EDA](#1-creating-credentials-in-ansible-eda)).
    - **Content-Type**:  
      Set the **Content-Type** header to `application/json` to ensure the payload is sent in JSON format.
7. Click **Submit** to save the REST message.
8. After saving, add an **HTTP Method** (typically POST):
    - Click **New** under the HTTP Methods related list.
    - Set the **HTTP Method** to `POST`.
    - Set the **Endpoint** to the Ansible EDA webhook receiver (e.g., `https://<eda-server>/endpoint`).
    - Save the HTTP method.
9. Test the REST message by using the **Test** feature to ensure it can successfully send data to your EDA endpoint.

![ServiceNow REST message](images/image5.png)

### 4. Create the Business Rule

The business rule will be used to automatically send an event to Ansible EDA whenever a new requested item is created in ServiceNow. This is the preferred method because, unlike flows—which require you to assign a flow for each catalog item—a business rule can act globally and trigger for any requested item. 

For the purposes of this demo, we will filter by catalog item name using a condition, so the business rule will only act when the requested item corresponds to the catalog item

1. Navigate to **System Definition > Business Rule** in ServiceNow.
2. Click **New** to create a new business rule.
3. Enter a **Name** for the business rule (e.g., "Send event to EDA").
4. Set the **Table** to **Requested Item [sc_req_item]**.
5. Set the **When to run** options:
    - Check **Insert** (so the rule runs when a new requested item is created).
    - Uncheck **Update** and other options if you only want this to trigger on creation.
6. Set the condition to match the catalog item name. For example:
   - **[Catalog Item] [is] [Provision New Virtual Machine]**
7. Set the **Advanced** option to true to enable scripting.
8. In the **Advanced** tab, add the following script that sends the REST message to EDA.

```javascript
(function executeRule(current, previous) {
	gs.info("Business Rule 'Send RITM to EDA' triggered for: " + current.number);
	var payload = {};
	var fields = new GlideRecordUtil().getFields(current);
	for(var field in fields) {
		var field_name = fields[field];
		var field_type = current.getElement(field_name).getED().getInternalType();
		if(field_name == 'variables') {
			continue;
		}
		else if(field_type == 'boolean' || field_type == 'journal_input') {
			var variable_display_value = current.getDisplayValue(field_name);
			if(variable_display_value) {
				payload[field_name] = variable_display_value;
			}
		}
		else {
			var variable_value = current.getValue(field_name);
			if(variable_value) {
				payload[field_name] = variable_value;
			}
		}
	}

	payload['variables'] = {};
    var variables = current.variables.getElements();
    for (var i=0; i<variables.length; i++) { 
        var question = variables[i].getQuestion();
		payload['variables'][question.getName()] = question.getValue();
    } 

	var REST_MESSAGE_NAME = "Send event to EDA";
	var request = new sn_ws.RESTMessageV2(REST_MESSAGE_NAME, "POST");
	var request_body = JSON.stringify(payload);
	request.setRequestBody(request_body);
	request.setTimeout(1000);
	request.execute();
})(current, previous);
```
9. Click **Submit** to save and activate the business rule.
10.  (Optional) Test the business rule by submitting a new catalog request and verifying that the event is sent to Ansible

![ServiceNow Business Rule](images/image6.png)