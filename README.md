**What is covered?**

* phpIPAM
* Catalog items
* Ansible
* Options
  * Forms
  * Inputs
  * REST option lists
  * Translation scripts
* File Templates

GitHub: [https://github.com/tyboyd02/phpIPAM](https://github.com/tyboyd02/phpIPAM)

**Goal**
Create a catalog item to easily deploy different versions of phpIPAM from a dynamic option list, through an automated process with an Ansible playbook.

**Adding the Ansible integration to Morpheus**
Start by going to integrations under Administration and add the repo that hosts the playbook. You can add the public repo provided above.

_Note: The ansible playbook and the Morpheus items have been exported to the repository above._

![image1](https://github.com/tyboyd02/phpIPAM/assets/121468777/094d0ffe-a4a1-47b0-8484-dd5d23841db3)

**Create a task**
This task will use the ansible playbook to install phpIPAM.
Under the Library Automation tab add a task.

Type: Ansible Playbook
Ansible Repo: Select the one you added in the previous step.
GIT REF: Main
Playbook: phpipaminstall.yml

![image2](https://github.com/tyboyd02/phpIPAM/assets/121468777/c359da2e-37e0-40b3-a1e6-8cc69199268e)

You will then create a workflow and add the newly created task in the post-Provision section. 

**File Template**

Next, we will take advantage of Morpheus file templates to automatically add the phpIPAM configuration. To do this go to File Templates section under the library templates.

File Name: phpipam.conf
File path: /etc/apache2/sites-available
Phase: Post Provision
Template:
```
<VirtualHost *:80>
    ServerAdmin admin@example.com
    DocumentRoot "/var/www/html/phpipam"
    ServerName <%=server.internalIp%>
    Redirect permanent / https://<%=server.internalIp%>
    <Directory "/var/www/html/phpipam">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog "/var/log/apache2/phpipam-error_log"
    CustomLog "/var/log/apache2/phpipam-access_log" combined
</VirtualHost>

<VirtualHost *:443>
    ServerAdmin adminexample.com
    DocumentRoot "/var/www/html/phpipam"
    ServerName <%=server.internalIp%>
    <Directory "/var/www/html/phpipam">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    SSLEngine on
    SSLCertificateFile /etc/ssl/phpipam.crt
    SSLCertificateKeyFile /etc/ssl/phpipam.key
    ErrorLog "/var/log/apache2/phpipam-error_log"
    CustomLog "/var/log/apache2/phpipam-access_log" combined
</VirtualHost>
```
Later we will add this to the phpIPAM node so it will automatically be added to the server when the catalog item is ordered. 

![image3](https://github.com/tyboyd02/phpIPAM/assets/121468777/af87ea31-2092-4068-a446-3fd70f644127)

**Option lists**

Next, we will make two option lists that will allow you to select what version of phpIPAM you want to be installed in the catalog item. Since phpIPAM has a version currently in development that you can install I first created an option list that lets you select between Development or Production install. Navigate to option lists under Library options.

Type: Manual
Dataset:
```
[
    {"name": "Dev", "value": "Dev"},
    {"name": "Prod", "value": "Prod"}
]
```
Following this create a dynamic option list that will pull available versions of phpIPAM. This will take advantage of the rest type and pull the tags available in phpIPAM GitHub.

Type: REST
Source URL: https://api.github.com/repos/phpipam/phpipam/tags

You will need to use a translation script to parse the JSON reply and to only add the version you want to be displayed. From my testing, my playbook only worked from version 1.3.2. Below is the translation script I used. 

```
var results = [];
for (var x = 0; x < data.length; x++) {
    var version = data[x].name; 
    if (version.localeCompare("v1.3.2") >= 0) {
        results.push({ name: version, value: data[x].tarball_url });
    }
}
```

**Form**

Next, create a form that meets your needs. I created one that allows them to select the name, group assignment, VMware clouds, resource pool, network, Prod or Dev install, version of phpIPAM, and the password for the phpIPAM MySQL user.

For my use case, it will only be used for quick testing, so I did not give them the ability to choose VM sizing within the forms. If this is needed feel free to add it.

_Note: The playbook provided requires Lifecycle state, version, and password to be passed as variables to complete._
* morpheus['customOptions']['lifecyclestage']
* morpheus['customOptions']['prodversions']
* morpheus['customOptions']['phpipammysqlpw']

![image4](https://github.com/tyboyd02/phpIPAM/assets/121468777/3ad2ff27-5fcc-4e07-a776-fba1493d81b5)

First, add an input to allow the user to select if they want a production or development install in the development stage. Choose the option list that we created earlier that let the user choose “Dev” or “Prod”

![image5](https://github.com/tyboyd02/phpIPAM/assets/121468777/99f9a232-1ec4-4cc3-82ba-d2a1e05a1769)

We will then want another input that becomes viewable if a prod install is selected that uses our phpIPAM version option list.

![image6](https://github.com/tyboyd02/phpIPAM/assets/121468777/3ec801a9-0e41-4359-b384-436c5af56295)

To make it become visible only when prod is selected in the previous option, we will take advantage of the advanced tab at the bottom. 
Visibility field: lifecyclestage:(Prod$)

We will also want to make it so it requires a version to be specified when prod is selected. 
Require filed: lifecyclestage:(Prod$)

![image7](https://github.com/tyboyd02/phpIPAM/assets/121468777/11421afd-896e-4427-9a51-cdb7bb3c9246)

 You will also have a section to allow what the phpIPAM mysql password will be. 

![image8](https://github.com/tyboyd02/phpIPAM/assets/121468777/3a3d54a9-3120-46ac-a0f3-3133a14692a2)

**Instance type**

Under Library, blueprints we will add an instance type to use for our catalog item.

Create an instance and a layout with the “install phpIPAM” workflow that we created earlier.

Then create a node within the layout and add the phpIPAM.conf file template here.<br>
![image9](https://github.com/tyboyd02/phpIPAM/assets/121468777/796123a4-8a72-4102-b4a3-0f6ff217aebf)

**Create a phpIPAM Catalog**

Go to Library, blueprints, catalog items, and click add. Create a new catalog item and fill out the information to your liking. Click Configure and choose the instance type we created earlier. Then follow the prompt and configure it to fit your needs.

For my purposes, I gave it 2 CPU cores with 8 GB of RAM and 200 GB of disk space.

Note: You do **not** need to select a workflow as it was added to the instance layout.

Next, add the form we created earlier.

For the instance name I wanted to be able to leave it blank to follow the naming policy or let them specify a name they wanted. To do this I added logic to the name in the config section.
![image10](https://github.com/tyboyd02/phpIPAM/assets/121468777/a08e5e4c-7a9c-4086-83d1-43ddaec29c93)

**Content**

I also wanted to have a nice home page with text and images that give the user an idea of what the catalog item provides with some additional information. In the Content section, I wrote a home page with markdown.

```
![Image](https://morpheusaio/tools/archives/download/Images/phpipam%20homepage.png)

## phpIPAM
phpIPAM is an open-source web IP address management application (IPAM). Its goal is to provide light IP address management. It is php-based application with MySQL database backend, using jQuery libraries, ajax and HTML5/CSS3 features.

[phpIPAM Releases](https://github.com/phpipam/phpipam/releases)

![Subnet Page](https://morpheusaio/tools/archives/download/Images/phpIPAMsubnet.png)

---

## Default Creds 
* Username: Admin
* Password: ipamadmin

---

## Dev branch: 
Use if you are comfortable using the development release of phpIPAM. 
[Github](https://github.com/phpipam/phpipam)
```
![image11](https://github.com/tyboyd02/phpIPAM/assets/121468777/2364a6a2-c625-4856-847c-d14d7b57adbb)

This uses Morpheus archives to host the images in a S3 bucket.
![image12](https://github.com/tyboyd02/phpIPAM/assets/121468777/570c6da7-1a8e-44ef-ab2e-e582b9e5e140)

Final product
![image13](https://github.com/tyboyd02/phpIPAM/assets/121468777/100e6e39-69df-4b76-99f1-dedcdfc62a10)
