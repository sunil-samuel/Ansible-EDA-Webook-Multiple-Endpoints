# Ansible Automation Platform (AAP) 2.4 - eda.webhook - Multiple Endpoints

A sample rulebook that uses eda webhooks to listen for events.  Given the endpoint to
which the event is sent, call different playbooks to process the event using `run_job_template`.
The code will delegate the events by sending the complete playload to the playbooks, which can
then do the processing.

```
NOTE:  There is only one listener on this hostname and port.  Hence, all endpoints would eventually
be received by one listener, but the rules will delegate the events to different `playbooks`
based on the endpoint to which the event was sent.
```

There are two ways to call a playbook within a rulebook:
1. `run_playbook` :: Only supported with the `ansible-rulebook` cli.  This will not
work with the EDA Controller UI.
1. `run_job_template` :: This is supported by the EDA Controller UI, but requires
some configuration (detailed below).

## Running Locally

The script can be tested locally prior to creating an activation within the EDA UI.
This is useful to debug the playbooks.

### Install Rulebook

This assumes that you already installed python3, pip, and ansible.

Install ansible-rulebook an ansible.eda as follows:

```sh
%> pip install ansible-rulebook
%> ansible-galaxy collection install ansible.eda
```
### Run the Rulebook to Listen for Events

use `ansible-rulebook` to run the code.  Using the `-v` parameter will produce more
debug logs.

```sh
%> ansible-rulebook --rulebook rulebooks/generic-webhook-run_playbook.yml -i inventory.yml --vars vars.yml

2023-08-15 10:30:57,459 - ansible_rulebook.app - INFO - Starting sources
 ...
2023-08-15 10:30:58 331 [drools-async-evaluator-thread] INFO org.drools.ansible.rulebook.integration.api.io.RuleExecutorChannel - Async channel connected
```

### Send JSON Payload to Rulebook

use `curl` or similar tool to send a `POST` payload to the listening host.

#### Call the `splunk` endpoint as follows:
--------------------------------------

```sh
curl -d '{"summary": "Test to create TI issue from mule","description": "Mule Testing Jira Api one level of Module","type": "Incident","priority": "3-Medium","reporter": "ag","moduleMapLevels": {"parent": "Common to All Modules"}, "moduleMapAssets": [{"name": "Rates | IRD"},{"name": "CRD | CRD"}]}' -H "Content-Type: application" -X POST http://localhost:5000/splunk
```

#### Call the `web` endpoint as follows:
-----------------------------------

```sh
curl -d '{"warn":"2023-08-16T05:01:48.101-04:00  WARN 111934 --- [http-nio-0.0.0.0-8082-exec-5] o.s.web.servlet.PageNotFound : No mapping for GET /somepage.html"}' -H "Content-Type: application" -X POST http://localhost:5000/web
```
---

## Project and Activation from AAP EDA UI

Once everything looks good, configure the AAP EDA UI to create a project and a activiation.  This assumes
that you already installed AAP 2.4 that supports EDA.

In order for the playbooks to be called by the main rulebook (configured within the EDA Controller UI), you
must configure two templates with the main AAP Controller UI, namely `eda-template-splunk` and `eda-template-web` that will
be called by the EDA rulebook.  Therefore, you will need to:

1. [Configure AAP Controller UI](#configure-aap-controller-ui)
1. [Configure EDA Controller UI](#configure-eda-controller-ui)

### Configure AAP Controller UI

On the AAP Controller UI, create:

#### New Organization (or Use Default Organization)
---

From menu, `Access → Organizations`.  Click on `Add`.  You can also use the Default organization.

#### New Application
---

From menu, `Administration → Applications`.  Click on `Add`.  Use the following values.

|Field       |Value        |
|------------|-------------|
|Name        |A unique name|
|Organization|Select the Organization|
|Authorization grant type|Select `Resource owner password-based`|
|Client type|Select `Public`|

#### Access Token to be Used by EDA Controller UI
---

From menu, `Access → Users`.  Click on the username (not the Action edit icon).  On the `Tokens` tab, click on `Add`.
Use the following values.

|Field       |Value        |
|------------|-------------|
|Application        | Click the search (magnifying glass icon) and select the Application you created |
|Scope|Select `Write`|

Click `Save`.  This will open a modal page with `Token Information`.  Copy and save the `Token` and `Refresh Token` value.
This token will be used to configure the EDA Controller UI.

```
NOTE:  This information will not be displayed again.
```
Make sure to save this information.

#### Project
---

Create a new `Project` pointing to the Git repostiory of our code.
From menu, `Resources → Projects`.  Click on `Add`.  Use the following values.

|Field       |Value        |
|------------|-------------|
|Organization        | Click the search (magnifying glass icon) and select the Organization |
|Source Control Type|Select `Git`|
|Source Control URL|`https://github.com/sunil-samuel/Ansible-EDA-Webook-Multiple-Endpoints`|

All others parameters, except for `Name` can be left blank.

#### Templates
---

Create two templates named `eda-template-splunk` and `eda-template-web`.
From menu, `Resources → Template`. Click `Add` and select `Add job template`.  Use the following values.

|Field       |Value        |
|------------|-------------|
| Name | `eda-template-splunk` |
|Job Typw        | `Run` |
|Inventory|Click the search (magnifying glass icon) and select the Inventory|
|Project|Click the search (magnifying glass icon) and select the Project|
|Playbook| Select `playbooks/splunk.yml`|
|Variables| Click the checkbox `Prompt on launch` |

Repeat for template `eda-template-web`.

To test these templates, from menu, `Resources → Templates`.  Clock on the rocket icon to launch this event.
On the modal page, select `YAML` and add the following into the `Variables` textarea and click `Next` and then `Launch`.

```yml
event:
  payload: hello
```

### Configure EDA Controller UI

On the EDA Controller UI:

#### Update the Controller Token
---

To allow the EDA UI the ability to call the templates we created on the AAP Controller UI, configure the token
that was created earlier in the [Access Token to be Used by EDA Controller UI](#access-token-to-be-used-by-eda-controller-ui) section.

From menu, `User Access → Users`, click on the username (Not the edit icon).  Click the `Controller Tokens` tab and
click on `+ Create controller token`.  Use the following values.

|Field       |Value        |
|------------|-------------|
| Token | The `Token` that was copied earlier (not the `Refresh Token`)  |

#### Create a New Project
---

From menu, `Resources → Projects`.  Click the `+ Create project`.  The current version of 2.4 supports only `Git`
for `SCM Type`.  Use the following values.

|Field       |Value        |
|------------|-------------|
| SCM URL | `https://github.com/sunil-samuel/Ansible-EDA-Webook-Multiple-Endpoints`  |

Following is an example:

![Create a Project](docs/01.create-project.png) 

#### Create EDA Activation
---

From menu, `Views → Rulebook Activations`.  Click the `+ Create rulebook activation`.  Use the following values.

|Field       |Value        |
|------------|-------------|
| Project | From the dropdown menu, select the project |
|Rulebook| From the dropdown menu, select `generic-webhook.yml` |
|Decision environment|select `Default Decision Environment` |

Following is an example:

![Create EDA Activation](docs/02.create-activations.png)

### Testing the Rulebook
---

Now that the EDA Controller UI and AAP Controller UI are working together, test that the payload is processed
correctly.

From menu, `Views → Rulebook Activations`.  Check that the activation is running.

![Validate Activation](docs/03.activation-running.png)

Click on the `name of the activation → History tab -> the name of the running activation`.

This will display the output.

Send a request to the URL using the correct IP address or hostname.  

```sh
[root@eda-controller-01 ~]# curl -d '{"summary": "Test to create TI issue from mule","description": "Mule Testing Jira Api one level of Module","type": "Incident","priority": "3-Medium","reporter": "ag","moduleMapLevels": {"parent": "Common to All Modules"}, "moduleMapAssets": [{"name": "Rates | IRD"},{"name": "CRD | CRD"}]}' -H "Content-Type: application" -X POST http://192.168.111.62:5000/splunk
```

Both AAP Controller and EDA Controller logs will display the execution of the rulebook and the template.