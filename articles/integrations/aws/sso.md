---
description: Learn how to use Single Sign-on (SSO) with AWS using the SAML2 Web App addon.
toc: true
topics:
  - integrations
  - aws
  - sso
contentType: how-to
useCase:
  - secure-an-api
  - integrate-third-party-apps
  - integrate-saas-sso
---

# Configure Single Sign-On with the AWS Console

By integrating Auth0 with AWS, you'll allow your users to log in to AWS using any supported [identity provider](/identityproviders). 

## Configure external Identity Provider in AWS

Set up an external identity provider in AWS using AWS's [Connect to your External Identity Provider](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html) doc--with one slight change. Rather than downloading the AWS metadata file, click **Show Individual Metadata Values**, and copy the **AWS SSO issuer URL** and **AWS SSO ACS URL**. You will use these in the next section.

Leave this page open in your browser, as you'll need to complete configuration in a future section.

## Configure Auth0

1. Log in to the [Auth0 Dashboard](${manage_url}/#/applications), and create a new [Application](/application) (you can also use an existing Application if you'd like). On the **Addons** tab, enable the **SAML2 Web App** addon.

    ![Applications](/media/articles/dashboard/guides/app-list.png)

2. When the configuration pop-up appears, on the **Settings** tab, populate **Application <dfn data-key="callback">Callback URL</dfn>** with `https://signin.aws.amazon.com/saml`.

  ![SAML2 Web App Settings](/media/articles/integrations/aws/configure.png)

Then paste the following <dfn data-key="security-assertion-markup-language">SAML</dfn> configuration code into **Settings**. Be sure to replace the AWS_SSO_ISSUER_URL and AWS_SSO_ACS_URL placeholders with the values you copied from AWS in the previous section.

```js
{
  "audience": "AWS_SSO_ISSUER_URL",
  "destination": "AWS_SSO_ACS_URL",
  "mappings": {
    "email": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
    "name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name"
  },
  "createUpnClaim": false,
  "passthroughClaimsWithNoMapping": false,
  "mapUnknownClaimsAsIs": false,
  "mapIdentities": false,
  "nameIdentifierFormat": "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress",
  "nameIdentifierProbes": [
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
  ]
}
```

3. Scroll to the bottom, and click **Enable**.

4. Click over to the **Usage** tab. You'll need to complete your AWS configuration of Auth0 as the external identity provider (IdP) in the next section, which requires you to provide the appropriate metadata to AWS. To download a file containing this information, click **Identity Provider Metadata**.

  ![SAML2 Web App Usage](/media/articles/integrations/aws/idp-download.png)

## Complete external Identity Provider configuration in AWS

Return to the AWS SSO page you left open during the first section, and upload the metadata file you downloaded and saved in the previous section. Review and Confirm that you are changing the identity source.

## Create an AWS IAM Role

To use the provider, you must create an IAM role using the provider in the role's trust policy.  

1. In the sidebar, under **Access Management**, navigate to **[Roles](https://console.aws.amazon.com/iam/home#/roles)**. Click **Create Role**.

2. On the next page, you will be asked to select the type of trusted entity. Select **SAML 2.0 Federation**. 

3. When prompted, set the provider you created above as the **SAML provider**. Select **Allow programmatic and AWS Management Console access**. Click **Next** to proceed.

4. On the **Attach Permission Policies** page, select the appropriate policies to attach to the role. These define the permissions that users granted this role will have with AWS. For example, to grant your users read-only access to IAM, filter for and select the `IAMReadOnlyAccess` policy. Once you are done, click **Next Step**.

5. The third **Create Role** screen is **Add Tags**. You can use tags to organize the roles you create if you will be creating a significant number of them.

6. On the **Review** page, set the **Role Name** and review your settings. Provide values for the following parameters:

  | Parameter | Definition | 
  | - | - |
  | Role name | A descriptive name for your role |
  | Role description | A description of what your role is used for |

7. Review the **Trusted entities** and **Policies** information, then click **Create Role**. At this point, you'll have created the necessary role to associate with your provider.

## Map the AWS Role to a User

::: note
For an example of defining a server-side rule that assigns a role in an advanced use case, see the [Amazon API Gateway tutorial](/integrations/aws-api-gateway).
:::

The **AWS roles** specified will be associated with an **IAM policy** that enforces the type of access allowed to a resource, including the AWS Consoles. To learn more about roles and policies, see [Creating IAM Roles](http://docs.aws.amazon.com/IAM/latest/UserGuide/roles-creatingrole.html).

To map an AWS role to a user, you'll need to create a [rule](/rules):

```js
function (user, context, callback) {

  user.awsRole = 'arn:aws:iam::951887872838:role/TestSAML,arn:aws:iam::951887872838:saml-provider/MyAuth0';
  user.awsRoleSession = user.name;

  context.samlConfiguration.mappings = {
    'https://aws.amazon.com/SAML/Attributes/Role': 'awsRole',
    'https://aws.amazon.com/SAML/Attributes/RoleSessionName': 'awsRoleSession'
  };

  callback(null, user, context);

}
```

In the code snippet above, `user.awsRole` identifies the AWS role and the IdP. The AWS role identifier comes before the comma, and the IdP identifier comes after the comma.

Your rule can obtain these two values in multiple ways. You can get these values from the IAM Console by selecting the items you created in AWS in the previous steps from the left sidebar. Both the Identity Provider and the Role you created have an ARN available to copy if you select them in the Console.

In the example above, both of these values are hard-coded into the rule. Alternatively, you might also store these values in the [user profile](/users/concepts/overview-user-profile) or derive them using other attributes. For example, if you're using Active Directory, you can map properties associated with users, such as group to the appropriate AWS role:

```js
var awsRoles = {
  'DomainUser': 'arn:aws:iam::951887872838:role/TestSAML,arn:aws:iam::95123456838:saml-provider/MyAuth0',
  'DomainAdmins': 'arn:aws:iam::957483571234:role/SysAdmins,arn:aws:iam::95123456838:saml-provider/MyAuth0'
};
user.awsRole = awsRoles[user.group];
user.awsRoleSession = user.email;

context.samlConfiguration.mappings = {
  'https://aws.amazon.com/SAML/Attributes/Role': 'awsRole',
  'https://aws.amazon.com/SAML/Attributes/RoleSessionName': 'awsRoleSession',
};
```

### Map Multiple Roles

You can also assign an array to the role mapping (so you'd have `awsRoles = [ role1, role2 ]` instead of `awsRoles: role1`)

For example, let's say that you have Active Directory Groups with the following structure:

```text
 var user = {
   app_metadata: {
     ad_groups: {
       "admins": "some info not aws related",
       "aws_dev_Admin": "arn:aws:iam::123456789111:role/Admin,arn:aws:iam::123456789111:saml-provider / Auth0",
       "aws_prod_ReadOnly": "arn:aws:iam::123456789999:role/ReadOnly,arn:aws:iam::123456789999:saml-provider / Auth0"
     }
   }
 };
```

Your rule might therefore looking something like this:

```js
function (user, context, callback) {

  var userGroups = user.app_metadata.ad_groups;

  function awsFilter(group) {
    return group.startsWith('aws_');
  }

  function mapGroupToRole(awsGroup) {
    return userGroups[awsGroup];
  }

  user.awsRole = Object.keys(userGroups).filter(awsFilter).map(mapGroupToRole);
  user.awsRoleSession = 'myawsuser'; // unique per user http://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html

  context.samlConfiguration.mappings = {
    'https://aws.amazon.com/SAML/Attributes/Role': 'awsRole',
    'https://aws.amazon.com/SAML/Attributes/RoleSessionName': 'awsRoleSession'
  };

  callback(null, user, context);

}
```

## Configure Session Expiration

If you want to extend the amount of time allowed to elapse before the AWS session expires (which is, by default, **3600 seconds**), you can do so using a custom [rule](/rules). Your rule sets the [**SessionDuration** attribute](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_saml_assertions.html) that changes the duration of the session.

```js
function (user, context, callback) {
    if(context.clientID !== 'YOUR_CLIENT_ID_HERE'){
      return callback(null, user, context);
    }

  user.awsRole = 'YOUR_ARN_HERE';
  user.awsRoleSession = 'YOUR_ROLE_SESSION_HERE';
  user.time = 1000; // time until expiration in seconds

  context.samlConfiguration.mappings = {
    'https://aws.amazon.com/SAML/Attributes/Role': 'YOUR-AWS-ROLE-NAME',
    'https://aws.amazon.com/SAML/Attributes/RoleSessionName': 'YOUR-AWS-ROLE-SESSION-NAME',
    'https://aws.amazon.com/SAML/Attributes/SessionDuration': 'time'   };

  callback(null, user, context);
}
```

## Test setup

You are now set up for Single Sign-on (SSO) to AWS and can test your setup.

1. Go to [Auth0 Dashboard > Application](${manage_url}/#/applications), and click the name of your application.
2. Click the **Addons** tab, and select the **SAML2 Web App** add-on.
3. Click the **Usage** tab.
4. Navigate to the **Identity Provider Login URL**. You should be redirected to the Auth0 login page. If you successfully sign in, you'll be redirected again--this time to AWS.
