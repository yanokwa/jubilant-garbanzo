FORMAT: 1A

# ODK Central API
[ODK Central Backend](https://github.com/opendatakit/central-backend) is a RESTful API server that provides key functionality for creating and managing Open Data Kit data collection campaigns. It couples with [Central Frontend](https://github.com/opendatakit/central-frontend), an independent frontend interface, to form [ODK Central](https://github.com/opendatakit/central), a complete user-installable ODK server solution. While Central Frontend is the primary consumer of the ODK Central API, the API this server provides is fully public and generic: anything that can be done in the user interface can be done directly via the API.

You can read on for a brief overview of the main concepts and how they fit together, or jump to one of the sections for a more in-depth description.

## API Overview

To use the API to manage your data collection campaigns, you will need to **authenticate** with it, so it knows who you are and what you are allowed to do. We provide multiple methods to authenticate with the API, as well as different models for managing the identities and permissions in your system. Human staff users that manage data collection campaigns are `User`s, and mobile devices are granted access via `Field Key`s, and each of these account types have their own way of authenticating. But, these concepts both boil down to `Actor`s, which are how the API actually thinks about authentication and permissioning.

In future versions of this server, the primary way an API consumer would authenticate with this server will be as a simple Actor through a standard OAuth 2.0 mechanism. That Actor can be granted a constrained set of rights to safely perform only all the actions it needs to. In this initial release, however, all Users are granted full administrative access, and password-based authentication as a User is the only way to gain full access to the system.

The `/users` resource can be used to create, manage, and delete **Users**. These are the staff members who have administrative rights on your server, some projects, or both. Additional tasks like resetting a user's password are also available. You could use this API to, for example, synchronize accounts with another system or mass-provision Users.

The rest of system is made up mostly of standard REST resources and subresources, nested under and partitioned by the `/projects` Projects resource. Forms, submissions to those forms, attachments on forms or submissions, and Field Keys ("App Users" in the management interface), are all subresources within `/projects`. This way, each project is essentially its own sandbox which can be managed and manipulated at will.

The `/projects/:id/app-users` subresource can be used to create, manage, and delete **Field Keys**.

The `/projects/:id/forms` resource and its subresource `/projects/:id/forms/…/submissions` provide full access to create, manage, and delete **`Form`s and `Submission`s to those Forms**. Each Form is a single ODK XForms survey, and many Submissions (filled-out forms, also sometimes called `Instance`s) may be attached to each Form. These resources are somewhat unique in that they are created by sending XML in the ODK XForms format instead of JSON. One can also retrieve all the multimedia attachments associated with any submission through the `/projects/:id/forms/…/submissions/…/attachments` subresource.

Forms and their submissions are also accessible through two **open standards specifications** that we follow:

* The [OpenRosa](https://docs.opendatakit.org/openrosa/) standard allows standard integration with tools like the [ODK Collect](https://docs.opendatakit.org/collect-intro/) mobile data collection app, or various other compatible tools like [Enketo](https://enketo.org/). It allows them to see the forms available on the server, and to send new submissions to them.
* The [OData](http://odata.org/) standard allows data to be shared between platforms for analysis and reporting. Tools like [Microsoft Power BI](https://powerbi.microsoft.com/en-us/) and [Tableau](https://public.tableau.com/en-us/s/) are examples of clients that consume the standard OData format and provide advanced features beyond what we offer. If you are looking for a straightforward JSON output of your data, or you are considering building a visualization or reporting tool, this is your best option.

Finally, **system configuration** is available via a set of specialized resources. Currently, the only available configuration allows you to set up data backups.

# Group Authentication

In ODK Central, the server thinks about identity and permissioning in terms of one core concept: the `Actor`. No matter how you authenticate with the API, you are doing so as an Actor of some kind or another, and when permissions are assigned and checked, they are done against the authenticated Actor.

In practice, there are two types of Actors available in the system today:

* `User`s are accounts used by the staff members who manage the server and the data collection campaigns. Presently, all Users have full administrative permissions to do anything in the system. They are the only account types that have passwords associated with them. They also always have an email address. Users can authenticate using **Session Bearer Tokens** or using **HTTPS Basic** authentication.
* `Field Key`s are only allowed to access the OpenRosa parts of the API: in essence, they are allowed to list forms, download form definitions, and create new submissions against those forms. They can only authenticate using **Field Key URL**s.

In a future version of the API, programmatic consumers will be more directly supported as their own Actor type, which can be granted limited permissions and can authenticate over **OAuth 2.0**.

Next, you will find documentation on each of the three authentication methods described above.

## Session Authentication [/v1/sessions]

This is the authentication method used by the ODK Central Frontend packaged with Central Backend. Only `User`s can authenticate this way. It consists mostly of two steps:

1. **Logging in**: presenting an Email Address and a Password for verification, after which a new `Session` is created. Associated with the Session is an expiration and a bearer token. Sessions expire 24 hours after they are created.
2. **Using the session**: each request to the API needs a header attached to it: `Authorization: Bearer {token}`. This authenticates that particular request as belonging to the Session we created by logging in.

You might notice that Step 2 greatly resembles how OAuth 2.0 works. This was an intentional first step towards OAuth support, and should make the forward migration of your code easier down the road.

### Logging in [POST]

In order to log a `User` in to a new `Session`, you must provide their credentials, in JSON format.

For security reasons, the only possible results are success or failure. No detail is provided upon failure.

Successful responses will come with an HTTP-Only, Secure-Only cookie. This cookie is primarily meant for use by the Central frontend, and we do not recommend relying upon it. It will only work on `GET` requests, and it will only work over HTTPS.

+ Attributes (object)
    + email (string, required) - The `User`'s full email address.
    + password (string, required) - The `User`'s password.


+ Request (application/json)
    + Body

            {
                "email": "my.email.address@opendatakit.org",
                "password": "my.super.secure.password"
            }

+ Response 200 (application/json)
    + Attributes (Session)

+ Response 401 (application/json)
    + Attributes (Error 401)

### Using the session [GET /v1/example]

Once you have logged in, to use your session token to authenticate with any action, supply it in a request header `Authorization` with a value of `Bearer {token}`, as seen here.

_(There is not really anything at `/v1/example`; this section only demonstrates how generally to use Session Bearer Token Authentication.)_

+ Request
    + Headers

            Authorization: Bearer lSpAIeksRu1CNZs7!qjAot2T17dPzkrw9B4iTtpj7OoIJBmXvnHM8z8Ka4QPEjR7

+ Response 200 (application/json)
    + Attributes (Success)

#### Logging out [DELETE /v1/sessions/{token}]

Logging out is not strictly necessary; all sessions expire 24 hours after they are created. But it can be a good idea, in case someone else manages to steal your token. To do so, issue a `DELETE` request to that token resource.

+ Parameters
    + token: `lSpAIeksRu1CNZs7!qjAot2T17dPzkrw9B4iTtpj7OoIJBmXvnHM8z8Ka4QPEjR7` (string, required) - The session bearer token, obtained at login time.

+ Request
    + Headers

            Authorization: Bearer lSpAIeksRu1CNZs7!qjAot2T17dPzkrw9B4iTtpj7OoIJBmXvnHM8z8Ka4QPEjR7

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

## HTTPS Basic Authentication [/v1/example]

Standard HTTP Basic Authentication is allowed, but **_strongly discouraged_**. This is because the server must verify your password with every single request, which is very slow to compute: typically, this will add hundreds of milliseconds to each request. For some one-off tasks and in cases where there is no other choice, it is reasonable to choose Basic authentication, but wherever possible we strongly encourage the use of any other authentication method.

In addition, because credentials are sent in plaintext as part of the request, **the server will only accept Basic auth over HTTPS**. If your ODK Central server is set up over plain HTTP, it will not accept Basic auth.

### Using Basic Authentication [GET]

To use HTTPS Basic Authentication, attach an `Authorization` header formatted so:

    `Authorization: Basic bXkuZW1haWwuYWRkcmVzc0BvcGVuZGF0YWtpdC5vcmc6bXkucGFzc3dvcmQ=`

As given by [the standard](https://en.wikipedia.org/wiki/Basic_access_authentication), the text following the `Basic` marker here is a base64 encoding of the credentials, provided in the form `email:password` (in this example `my.email.address@opendatakit.org:my.password`).

Unlike the standard, we do not require the client to first send an unauthenticated request and retry the request only after receiving a `WWW-Authenticate` response, and in fact we will never send the `WWW-Authenticate` header. This is mostly because, as noted above, we generally discourage the use of this authentication method, and would rather not advertise its use openly. As a result, if you wish to use Basic Authentication, directly supply the header on any request that needs it.

_(There is not really anything at `/v1/example`; this section only demonstrates how generally to use Basic Authentication.)_

+ Request
    + Headers

            Authorization: Basic bXkuZW1haWwuYWRkcmVzc0BvcGVuZGF0YWtpdC5vcmc6bXkucGFzc3dvcmQ=

+ Response 200 (application/json)
    + Attributes (Success)

## Field Key Authentication [/v1/key/{fieldKey}/example]

Field keys are only allowed to list and download forms, and upload new submissions to those forms. Primarily, this is to allow clients like ODK Collect to use the OpenRosa API (`/formList` and `/submission`), but any action in this API reference falling into those categories will be allowed.

+ Parameters
    + `fieldKey`: `!Ms7V3$Zdnd63j5HFacIPFEvFAuwNqTUZW$AsVOmaQFf$vIC!F8dJjdgiDnJXXOt` (string, required) - The field key token. As with Session Bearer tokens, these tokens only contain URL-safe characters, so no escaping is required.

### Using Field Key Authentication [GET]

To use Field Key Authentication, first obtain a Field Key, typically by using the configuration panel in the user interface, or else by using the [Field Key API Resource(/reference/accounts-and-users/app-users). Once you have the token, you can apply it to any eligible action by prefixing the URL with `/key/{fieldKey}` as follows:

    `/v1/key/!Ms7V3$Zdnd63j5HFacIPFEvFAuwNqTUZW$AsVOmaQFf$vIC!F8dJjdgiDnJXXOt/example/request/path`

_(There is not really anything at `/v1/example`; this section only demonstrates how generally to use Field Key Authentication.)_

+ Response 200 (application/json)
    + Attributes (Success)

### Revoking a Field Key [DELETE /v1/sessions/{token}]

The token associated with a Field Key is actually just its Session Token. As a result, although a Field Key Token can uniquely be used as a URL prefix as described here, the session associated with it can be revoked in exactly the same way a session is logged out, by issuing a `DELETE` request to its Session resource.

Note, however, that a Field Key cannot revoke itself; a `User` must perform this action.

+ Parameters
    + token: `lSpAIeksRu1CNZs7!qjAot2T17dPzkrw9B4iTtpj7OoIJBmXvnHM8z8Ka4QPEjR7` (string, required) - The session bearer token, obtained at login time.

+ Request
    + Headers

            Authorization: Bearer lSpAIeksRu1CNZs7!qjAot2T17dPzkrw9B4iTtpj7OoIJBmXvnHM8z8Ka4QPEjR7

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

# Group Accounts and Users

Today, there are two types of accounts: `Users`, which are the administrative accounts held by staff members managing the data collection process, and `Field Keys`, which are restricted access keys granted per Project to data collection clients in the field. Although both of these entities are backed by `Actor`s as we explain in the [Authentication section](/reference/authentication) above, there is not yet any way to directly create or manipulate an Actor. Today, you can only create, manage, and delete Users and Field Keys.

## Users [/v1/users]

Presently, it is possible to create and list `User`s in the system, as well as to perform password reset operations. In future versions of this API it will be possible to manage existing user information and delete accounts as well.

### Listing all Users [GET]

Currently, there are no paging or filtering options, so listing `User`s will get you every User in the system, every time.

+ Response 200 (application/json)
    + Attributes (array[User])

+ Response 403 (application/json)
    + Attributes (Error 403)

### Creating a new User [POST]

All that is required to create a new user is an email address. That email address will receive a message instructing the new user on how to claim their new account and set a password.

Optionally, a password may also be supplied as a part of this request. If it is, the account is immediately usable with the given credentials. However, an email will still be dispatched with claim instructions as above.

+ Request (application/json)
    + Attributes
        + email: `my.email.address@opendatakit.org` (string, required) - The email address of the User account to be created.
        + password: `my.super.secure.password` (string, optional) - If provided, the User account will be created with this password. Otherwise, the user will still be able set their own password later.

    + Body

            { "email": "my.email.address@opendatakit.org" }

+ Response 200 (application/json)
    + Attributes (User)

+ Response 400 (application/json)
    + Attributes (Error 400)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Getting user details [GET /v1/users/{actorId}]

Typically, you supply the integer ID to get information about the user associated with that id.

However, if you only have a Bearer token, for example, you don't have any information about the user attached to that session, including even the ID with which to get more information. So you can instead supply the text `current` to get the user information associated with the authenticated session.

+ Parameters
    + actorId: `42` (string, required) - Typically the integer ID of the `User`. For getting user details, you can also supply the text `current`, which will tell you about the currently authenticated user.

+ Response 200 (application/json)
    + Attributes (User)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Modifying a User [PATCH /v1/users/{actorId}]

You can `PATCH` JSON data to update User details. Not all user information is modifiable; right now, the following fields may be updated:

* `displayName` sets the friendly display name the web interface uses to refer to the user.
* `email` sets the email address associated with the account.

When user details are updated, the `updatedAt` field will be automatically updated.

+ Parameters
    + actorId: `42` (string, required) - The integer ID of the `User`.

+ Request (application/json)
    + Attributes
        + displayName: `New Name` (string, optional) - The friendly display name that should be associated with this User.
        + email: `new.email.address@opendatakit.org` (string, optional) - The email address that should be associated with this User.

    + Body

            {
              "displayName": "New Name",
              "email": "new.email.address@opendatakit.org"
            }

+ Response 200 (application/json)
    + Attributes (User)

+ Response 400 (application/json)
    + Attributes (Error 400)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Directly updating a user password [PUT /v1/users/{actorId}/password]

To directly update a user password, you will need to reprove the user's intention by supplying the `old` password alongside the `new`. If you simply want to initiate an email-based password reset process, see the following endpoint.

+ Parameters
    + actorId: `42` (string, required) - The integer ID of the `User`.

+ Request (application/json)
    + Attributes
        + old: `old.password` (string, required) - The user's current password.
        + new: `new.password` (string, required) - The new password that the user wishes to set.

    + Body

            {
              "old": "old.password",
              "new": "new.password"
            }

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Initating a password reset [POST /v1/users/reset/initiate{?invalidate}]

Anybody can initate a reset of any user's password. An email will be sent with instructions on how to complete the password reset; it contains a token that is required to complete the process.

The optional query parameter `invalidate` may be set to `true` to immediately invalidate the user's current password, regardless of whether they complete the reset process. This can be done if, for example, their password has been compromised. In order to do this, though, the request must be performed as an authenticated user with permission to do this. If invalidation is attempted without the proper permissions, the entire request will fail.

If the email address provided does not match any user in the system, that address will still be sent an email informing them of the attempt and that no account was found.

+ Parameters
    + invalidate: `true` (boolean, optional) - Specify `true` in order to immediately invalidate the user's present password.

+ Request (application/json)
    + Attributes
        + email: `my.email.address@opendatakit.org` (string, required) - The email address of the User account whose password is to be reset.

    + Body

            { "email": "my.email.address@opendatakit.org" }

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

## Field Keys [/v1/projects/{projectId}/app-users]

Field Keys may only be created, fetched, and manipulated within the nested Projects subresource, as Field Keys themselves are limited to the Project in which they are created. Through the `Field Key`s API, you can create, list, and delete the Field Keys of any given Project. Because they have extremely limited permissions, Field Keys cannot manage themselves; only `User`s may access this API.

For more information about the `/projects` containing resource, please see the following section.

+ Parameters
    + projectId: `7` (number, required) - The numeric ID of the Project

### Listing all Field Keys [GET]

Currently, there are no paging or filtering options, so listing `Field Key`s will get you every Field Key in the system, every time.

This endpoint supports retrieving extended metadata; provide a header `X-Extended-Metadata: true` to additionally retrieve the `lastUsed` timestamp of each Field Key, as well as to inflate the `createdBy` from an `Actor` ID reference to an actual Actor metadata object.

+ Response 200 (application/json)
    This is the standard response, if Extended Metadata is not requested:

    + Attributes (array[Field Key])

+ Response 200 (application/json; extended)
    This is the Extended Metadata response, if requested via the appropriate header:

    + Attributes (array[Extended Field Key])

+ Response 403 (application/json)
    + Attributes (Error 403)

### Creating a new Field Key [POST]

The only information required to create a new `Field Key` is its `displayName` (this is called "Nickname" in the administrative panel).

+ Request (application/json)
    + Attributes
        + displayName: `My Display Name` (string, required) - The friendly nickname of the `Field Key` to be created.

    + Body

            { "displayName": "My Display Name" }

+ Response 200 (application/json)
    + Attributes (Field Key)

+ Response 400 (application/json)
    + Attributes (Error 400)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Deleting a Field Key [DELETE /v1/projects/{projectId}/app-users/{id}]

You don't have to delete a `Field Key` in order to cut off its access. Using a `User`'s credentials you can simply [log the Field Key's session out](/reference/authentication/field-key-authentication/revoking-a-field-key) using its token. This will end its session without actually deleting the Field Key, which allows you to still see it in the configuration panel and inspect its history. This is what the administrative panel does when you choose to "Revoke" the Field Key.

That said, if you do wish to delete the Field Key altogether, you can do so by issuing a `DELETE` request to its resource path. Field Keys cannot delete themselves.

+ Parameters
    + id: `16` (number, required) - The numeric ID of the Field Key

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

# Group Project Management

Apart from staff users ("Web Users" in the Central management interface) and some site-wide configuration details like backups, all of ODK Central's objects (Forms, Submissions, Field Keys) are partitioned by Project, and available only as subresources below the main Projects resource.

## Projects [/v1/projects]

_(introduced: version 0.4.0)_

You must create a containing Project before you can create any of its subobjects.

### Listing all Projects [GET]

Currently, there are no paging or filtering options, so listing `Project`s will get you every Project in the system, every time.

This endpoint supports retrieving extended metadata; provide a header `X-Extended-Metadata: true` to additionally retrieve the `forms` count of forms within the Project, as well as the `lastSubmission` timestamp of the latest submission to any for in the project, if any.

+ Response 200 (application/json)
    This is the standard response, if Extended Metadata is not requested:

    + Attributes (array[Project])

+ Response 200 (application/json; extended)
    This is the Extended Metadata response, if requested via the appropriate header:

    + Attributes (array[Extended Project])

+ Response 403 (application/json)
    + Attributes (Error 403)

### Creating a Project [POST /v1/projects]

To create a Project, the only information you must supply (via POST body) is the desired name of the Project.

+ Request (application/json)
    + Attributes
        + name: `Project Name` (string, required) - The desired name of the Project.

    + Body

            { "name": "Project Name" }

+ Response 200 (application/json)
    + Attributes (Project)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Getting Project Details [GET /v1/projects/{id}]

To get just the details of a single Project, `GET` its single resource route by its numeric ID.

This endpoint supports retrieving extended metadata; provide a header `X-Extended-Metadata: true` to additionally retrieve the `forms` count of forms within the Project, as well as the `lastSubmission` timestamp of the latest submission to any for in the project, if any.

+ Parameters
    + id: `16` (number, required) - The numeric ID of the Project

+ Response 200 (application/json)
    This is the standard response, if Extended Metadata is not requested:

    + Attributes (Project)

+ Response 200 (application/json; extended)
    This is the Extended Metadata response, if requested via the appropriate header:

    + Attributes (Extended Project)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Updating Project Details [PATCH /v1/projects/{id}]

Currently, the only Project information that may be updated is its name.

+ Parameters
    + id: `16` (number, required) - The numeric ID of the Project

+ Request (application/json)
    + Attributes
        + name: `New Project Name` (string, required) - The desired name of the Project.

    + Body

            { "name": "New Project Name" }

+ Response 200 (application/json)
    + Attributes (Project)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Deleting a Project [DELETE /v1/projects/{id}]

Deleting a Project will remove it from the management interface and make it permanently inaccessible. Do not do this unless you are certain you will never need any of its data again.

+ Parameters
    + id: `16` (number, required) - The numeric ID of the Project

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

# Group Forms and Submissions

`Form`s are the heart of ODK. They are created out of XML documents in the [ODK XForms](https://opendatakit.github.io/xforms-spec/) specification format. The [Intro to Forms](https://docs.opendatakit.org/form-design-intro/) on the ODK Documentation website is a good resource if you are unsure what this means. Once created, Forms can be retrieved in a variety of ways, their state can be managed, and they can be deleted.

`Submission`s are filled-out forms (also called `Instance`s in some other ODK documentation). Each is associated with a particular Form (and in many cases with a particular _version_ of a Form), and is also created out of a standard XML format based on the Form itself. Submissions can be sent with many accompanying multimedia attachments, such as photos taken in the course of the survey. Once created, the Submissions themselves as well as their attachments can be retrieved through this API.

These subsections cover only the modern RESTful API resources involving Forms and Submissions. For documentation on the OpenRosa endpoints (which can be used to list Forms and submit Submissions), or the OData endpoints (which can be used to bulk-export the data in the standardize OData format), see those sections below.

## Forms [/v1/projects/{projectId}/forms]

In this API, `Form`s are distinguished by their [`formId`](https://opendatakit.github.io/xforms-spec/#primary-instance)s, which are a part of the XForms XML that defines each Form. In fact, as you will see below, many of the properties of a Form are extracted automatically from the XML: `hash`, `name`, `version`, as well as the `formId` itself (which to reduce confusion internally is known as `xmlFormId` in ODK Central).

The only other property Forms currently have is `state`, which can be used to control whether Forms show up in mobile clients like ODK Collect for download, as well as whether they accept new `Submission`s or not.

It is not yet possible to modify a Form's XML definition once it is created.

+ Parameters
    + projectId: `16` (number, required) - The numeric ID of the Project

### List all Forms [GET]

Currently, there are no paging or filtering options, so listing `Form`s will get you every Form in the system, every time.

This endpoint supports retrieving extended metadata; provide a header `X-Extended-Metadata: true` to additionally retrieve the `submissions` count of the number of `Submission`s that each Form has, as well as the `lastSubmission` most recent submission timestamp and the actual `xml` XForms XML form definition backing the form.

+ Response 200 (application/json)
    This is the standard response, if Extended Metadata is not requested:

    + Attributes (array[Form])

+ Response 200 (application/json; extended)
    This is the Extended Metadata response, if requested via the appropriate header:

    + Attributes (array[Extended Form])

+ Response 403 (application/json)
    + Attributes (Error 403)

### Creating a new Form [POST]

When creating a `Form`, the only required data is the actual XForms XML itself. Use it as the `POST` body with a `Content-Type` header of `application/xml` (`text/xml` works too), and the Form will be created.

If the combination of (`xmlFormId`, `version`) conflict with any existing Form, current or deleted, the request will be rejected. We consider even deleted forms when enforcing this restriction to prevent confusion in case a survey client already has the other version of that Form downloaded.

The API will currently check the XML's structure in order to extract the information we need about it, but ODK Central does _not_ run comprehensive validation on the full contents of the XML to ensure compliance with the ODK specification. Future versions will likely do this, but in the meantime you will have to use a tool like [ODK Validate](https://opendatakit.org/use/validate/) to be sure your Forms are correct.

+ Request (application/xml)
    + Body

            <h:html xmlns="http://www.w3.org/2002/xforms" xmlns:h="http://www.w3.org/1999/xhtml" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:jr="http://openrosa.org/javarosa">
              <h:head>
                <h:title>Simple</h:title>
                <model>
                  <instance>
                    <data id="simple" version="2.1">
                      <meta>
                        <instanceID/>
                      </meta>
                      <name/>
                      <age/>
                    </data>
                  </instance>

                  <bind nodeset="/data/meta/instanceID" type="string" readonly="true()" calculate="concat('uuid:', uuid())"/>
                  <bind nodeset="/data/name" type="string"/>
                  <bind nodeset="/data/age" type="int"/>
                </model>

              </h:head>
              <h:body>
                <input ref="/data/name">
                  <label>What is your name?</label>
                </input>
                <input ref="/data/age">
                  <label>What is your age?</label>
                </input>
              </h:body>
            </h:html>

+ Response 200 (application/json)
    + Attributes (Form)

+ Response 400 (application/json)
    + Attributes (Error 400)

+ Response 403 (application/json)
    + Attributes (Error 403)

### › Individual Form [/v1/projects/{projectId}/forms/{xmlFormId}]

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.

#### Getting Form Details [GET]

This endpoint supports retrieving extended metadata; provide a header `X-Extended-Metadata: true` to additionally retrieve the `submissions` count of the number of `Submission`s that this Form has, as well as the `lastSubmission` most recent submission timestamp and the actual `xml` XForms XML form definition backing the form.

+ Response 200 (application/json)
    This is the standard response, if Extended Metadata is not requested:

    + Attributes (Form)

+ Response 200 (application/json; extended)
    This is the Extended Metadata response, if requested via the appropriate header:

    + Attributes (Extended Form)

+ Response 403 (application/json)
    + Attributes (Error 403)

#### Retrieving Form XML [GET /v1/projects/{projectId}/forms/{xmlFormId}.xml]

To get only the XML of the `Form` rather than all of the details with the XML as one of many properties, just add `.xml` to the end of the request URL. This endpoint primarily exists so that OpenRosa survey clients have a way to directly retrieve the XML.

+ Response 200 (application/xml)
    + Body

            <h:html xmlns="http://www.w3.org/2002/xforms" xmlns:h="http://www.w3.org/1999/xhtml" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:jr="http://openrosa.org/javarosa">
              <h:head>
                <h:title>Simple</h:title>
                <model>
                  <instance>
                    <data id="simple" version="2.1">
                      <meta>
                        <instanceID/>
                      </meta>
                      <name/>
                      <age/>
                    </data>
                  </instance>

                  <bind nodeset="/data/meta/instanceID" type="string" readonly="true()" calculate="concat('uuid:', uuid())"/>
                  <bind nodeset="/data/name" type="string"/>
                  <bind nodeset="/data/age" type="int"/>
                </model>

              </h:head>
              <h:body>
                <input ref="/data/name">
                  <label>What is your name?</label>
                </input>
                <input ref="/data/age">
                  <label>What is your age?</label>
                </input>
              </h:body>
            </h:html>

+ Response 403 (application/json)
    + Attributes (Error 403)

#### Retrieving Form Schema JSON [GET /v1/projects/{projectId}/forms/{xmlFormId}.schema.json{?flatten}]

_(introduced: version 0.2.0)_

For applications that do not rely on JavaRosa, it can be challenging to parse XForms XML into a simple schema structure. Because Central Backend already implements and performs such an operation for its own internal purposes, we also expose this utility for any downstream consumers which wish to make use of it.

While this may eventually overlap with the new OData JSON CSDL specification, we are likely to maintain this API as it more closely mirrors the original XForms data types and structure.

By default, XForms groups are represented as tree structures with children, just like repeats. However, for many purposes (such as outputting a table), this is extra homework. Once again, Central Backend already has a utility to make this easier; add the querystring parameter `?flatten=true` to flatten groups. This will give you a `path`-based field accessor scheme instead of a `name`-based one (see the sample output for details).

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.
    + flatten: `false` (boolean, optional) - If set to `true`, will flatten groups.

+ Response 200 (application/json)
    + Body

            [{
              name: 'meta', type: 'structure',
              children: [{ name: 'instanceID', type: 'string' }]
            }, {
              name: 'name', type: 'string',
            }, {
              name: 'age', type: 'int',
            }]

+ Response 200 (application/json; ?flatten=true)
    + Body

            [
              { path: [ 'meta', 'instanceID' ], type: 'string' },
              { path: [ 'name' ], type: 'string' },
              { path: [ 'age' ], type: 'int' }
            ]

+ Response 403 (application/json)
    + Attributes (Error 403)

#### Modifying a Form [PATCH]

It is currently possible to modify two things about a `Form`: its `name`, which is its friendly display name, and its `state`, which governs whether it is available for download onto survey clients and whether it accepts new `Submission`s. See the `state` Attribute in the Request documentation to the right to see the possible values and their meanings.

We use `PATCH` rather than `PUT` to represent the update operation, so that you only have to supply the properties you wish to change. Anything you do not supply will remain untouched.

+ Request (application/json)
    + Attributes
        + name: `A New Name` (string, optional) - If supplied, the Form friendly name will be updated to this value.
        + state (Form State, optional) - If supplied, the Form lifecycle state will move to this value.

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 400 (application/json)
    + Attributes (Error 400)

+ Response 403 (application/json)
    + Attributes (Error 403)

#### Deleting a Form [DELETE]

Only `DELETE` a `Form` if you are sure you will never need it again. If your goal is to prevent it from showing up on survey clients like ODK Collect, consider setting its `state` to `closing` or `closed` instead (see [Modifying a Form](/reference/forms-and-submissions/'-individual-form/modifying-a-form) just above for more details).

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

### › Form Attachments [/v1/projects/{projectId}/forms/{xmlFormId}/attachments]

Form Attachments for each form are automatically determined when the form is first created, by scanning the XForms definition for references to media or data files. Because of this, it is not possible to directly modify the list of form attachments; that list is fully determined by the given XForm. Instead, the focus of this API subresource is around communicating that expected list of files, and uploading binaries into those file slots.

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.

#### Listing expected Form Attachments [GET]

As mentioned above, the list of expected form attachments is determined at form creation time, from the XForms definition. This endpoint allows you to fetch that list of expected files, and will tell you whether the server is in possession of each file or not.

This endpoint supports retrieving extended metadata; provide a header `X-Extended-Metadata: true` to additionally fetch a `updatedAt` field indicating the last time each file's binary content was modified (set or cleared).

+ Response 200 (application/json)
    This is the standard response, if Extended Metadata is not requested:

    + Attributes (array[Form Attachment])

+ Response 200 (application/json; extended)
    This is the Extended Metadata response, if requested via the appropriate header:

    + Attributes (array[Extended Form Attachment])

+ Response 403 (application/json)
    + Attributes (Error 403)

#### Downloading a Form Attachment [GET /v1/projects/{projectId}/forms/{xmlFormId}/attachments/{filename}]

To download a single file, use this endpoint. The appropriate `Content-Disposition` (attachment with a filename) and `Content-Type` (based on the type supplied at upload time) will be given.

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.

+ Response 200
    + Headers

            Content-Type: {the MIME type of the attachment file itself}
            Content-Disposition: attachment; filename={the file's name}

    + Body

            (binary data)

+ Response 403 (application/json)
    + Attributes (Error 403)

#### Uploading a Form Attachment [POST /v1/projects/{projectId}/forms/{xmlFormId}/attachments/{filename}]

To upload a binary to an expected file slot, `POST` the binary to its endpoint. Supply a `Content-Type` MIME-type header if you have one.

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.

+ Request (*/*)
    + Body

            (binary data)

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

#### Clearing a Form Attachment [DELETE /v1/projects/{projectId}/forms/{xmlFormId}/attachments/{filename}]

Because Form Attachments are completely determined by the XForms definition of the form itself, there is no direct way to entirely remove a Form Attachment entry from the list, only to clear its uploaded content. Thus, when you issue a `DELETE` to the attachment's endpoint, that is what happens.

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

## Submissions [/v1/projects/{projectId}/forms/{xmlFormId}/submissions]

`Submission`s are available as a subresource under `Form`s. So, for instance, `/v1/projects/1/forms/myForm/submissions` refers only to the Submissions that have been submitted to the Form `myForm`.

Once created (which, like with Forms, is done by way of their XML data rather than a JSON description), it is possible to retrieve and export Submissions in a number of ways, as well as to access the multimedia `Attachment`s associated with each Submission.

+ Parameters
    + projectId: `16` (number, required) - The numeric ID of the Project
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.

### Listing all Submissions on a Form [GET]

Currently, there are no paging or filtering options, so listing `Submission`s will get you every Submission in the system, every time.

This endpoint supports retrieving extended metadata; provide a header `X-Extended-Metadata: true` to inflate the `submitter` from an `Actor` ID reference to an actual Actor metadata object and return the actual `xml` submission XML data.

+ Response 200 (application/json)
    This is the standard response, if Extended Metadata is not requested:

    + Attributes (array[Submission])

+ Response 200 (application/json; extended)
    This is the Extended Metadata response, if requested via the appropriate header:

    + Attributes (array[Extended Submission])

+ Response 403 (application/json)
    + Attributes (Error 403)

### Exporting Form Submissions to CSV [GET /v1/projects/{projectId}/forms/{xmlFormId}/submissions.csv.zip]

To export all the `Submission` data associated with a `Form`, just add `.csv.zip` to the end of the listing URL. The response will be a ZIP file containing one or more CSV files, as well as all multimedia attachments associated with the included Submissions.

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.

+ Response 200
    + Headers

            Content-Disposition: attachment; filename=simple.zip

    + Body

            (binary data)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Getting Submission details [GET /v1/projects/{projectId}/forms/{xmlFormId}/submissions/{instanceId}]

Like how `Form`s are addressed by their XML `formId`, individual `Submission`s are addressed in the URL by their `instanceId`.

This endpoint supports retrieving extended metadata; provide a header `X-Extended-Metadata: true` to inflate the `submitter` from an `Actor` ID reference to an actual Actor metadata object and return the actual `xml` submission XML data.

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.
    + instanceId: `uuid:85cb9aff-005e-4edd-9739-dc9c1a829c44` (string, required) - The `instanceId` of the Submission being referenced.

+ Response 200 (application/json)
    This is the standard response, if Extended Metadata is not requested:

    + Attributes (Submission)

+ Response 200 (application/json; extended)
    This is the Extended Metadata response, if requested via the appropriate header:

    + Attributes (Extended Submission)

+ Response 403 (application/json)
    + Attributes (Error 403)

#### Retrieving Submission XML [GET /v1/projects/{projectId}/forms/{xmlFormId}/submissions/{instanceId}.xml]

To get only the XML of the `Submission` rather than all of the details with the XML as one of many properties, just add `.xml` to the end of the request URL.

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.
    + instanceId: `uuid:85cb9aff-005e-4edd-9739-dc9c1a829c44` (string, required) - The `instanceId` of the Submission being referenced.

+ Response 200 (application/xml)
    + Body

            <data id="simple">
              <orx:meta><orx:instanceID>uuid:85cb9aff-005e-4edd-9739-dc9c1a829c44</orx:instanceID></orx:meta>
              <name>Alice</name>
              <age>32</age>
            </data>

## Attachments [/v1/projects/{projectId}/forms/{xmlFormId}/submissions/{instanceId}/attachments]

When a `Submission` is created, either over the OpenRosa or the REST interface, its XML data is analyzed to determine which file attachments it references: these may be photos or video taken as part of the survey, or an audit/timing log, among other things. Each reference is an expected attachment, and these expectations are recorded permanently alongside the Submission.

With this subresource, you can list the expected attachments, see whether the server actually has a copy or not, and download, upload, re-upload, or clear binary data for any particular attachment.

+ Parameters
    + xmlFormId: `simple` (string, required) - The `xmlFormId` of the Form being referenced.
    + instanceId: `uuid:85cb9aff-005e-4edd-9739-dc9c1a829c44` (string, required) - The `instanceId` of the Submission being referenced.

### Listing expected Submission Attachments [GET]

 You can retrieve the list of expected Submission attachments at this route, along with a boolean flag indicating whether the server actually has a copy of the expected file or not. If the server has a file, you can then append its filename to the request URL to download only that file (see below).

+ Response 200 (application/json)
    + Attributes (array[Submission Attachment])

    + Body

            [{
                "name": "file1.jpg",
                "exists": true
            }, {
                "name": "file2.jpg",
                "exists": false
            }, {
                "name": "file3.jpg",
                "exists": true
            }]

+ Response 403 (application/json)
    + Attributes (Error 403)

### Downloading an Attachment [GET /v1/projects/{projectId}/forms/{xmlFormId}/submissions/{instanceId}/attachments/{filename}]

The `Content-Type` and `Content-Disposition` will be set appropriately based on the file itself when requesting an attachment file download.

+ Parameters
    + filename: `file1.jpg` (string, required) - The name of the file as given by the Attachments listing resource.

+ Response 200
    + Headers

            Content-Type: {the MIME type of the attachment file itself}
            Content-Disposition: attachment; filename={the file's name}

    + Body

            (binary data)

+ Response 403 (application/json)
    + Attributes (Error 403)

## Uploading an Attachment [POST /v1/projects/{projectId}/forms/{xmlFormId}/submissions/{instanceId}/attachments/{filename}]

_(introduced: version 0.4.0)_

To upload a binary to an expected file slot, `POST` the binary to its endpoint. Supply a `Content-Type` MIME-type header if you have one.

+ Parameters
    + filename: `file1.jpg` (string, required) - The name of the file as given by the Attachments listing resource.

+ Request (*/*)
    + Body

            (binary data)

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

## Clearing a Submission Attachment [DELETE /v1/projects/{projectId}/forms/{xmlFormId}/submissions/{instanceId}/attachments/{filename}]

_(introduced: version 0.4.0)_

Because Submission Attachments are completely determined by the XML data of the submission itself, there is no direct way to entirely remove a Submission Attachment entry from the list, only to clear its uploaded content. Thus, when you issue a `DELETE` to the attachment's endpoint, that is what happens.

+ Parameters
    + filename: `file1.jpg` (string, required) - The name of the file as given by the Attachments listing resource.

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

# Group OpenRosa Endpoints

[OpenRosa](https://bitbucket.org/javarosa/javarosa/wiki/OpenRosaAPI) is an API standard which accompanies the ODK XForms XML standard, allowing compliant servers and clients to use a common protocol to communicate `Form`s and `Submission`s to each other. When survey clients like ODK Collect and Enketo submit Submission data to a Form, this is the API they use.

ODK Central is _not_ a fully compliant OpenRosa server. OpenRosa requires compliance with five major components:

1. [**Metadata Schema**](https://bitbucket.org/javarosa/javarosa/wiki/OpenRosaMetaDataSchema), which defines a standard way to include metadata like the survey device ID and survey duration with a Submission. ODK Central will accept and return this data, but does nothing special with anything besides the `instanceId` at this time.
2. [**HTTP Request API**](https://bitbucket.org/javarosa/javarosa/wiki/OpenRosaRequest), which defines a set of requirements every OpenRosa request and response must follow. ODK Central is fully compliant with this component.
3. [**Form Submission API**](https://bitbucket.org/javarosa/javarosa/wiki/FormSubmissionAPI), which defines how Submissions are submitted to the server. ODK Central is fully compliant with this component.
4. [**Authentication API**](https://bitbucket.org/javarosa/javarosa/wiki/AuthenticationAPI), which defines how users authenticate with the server. ODK Central provides [three authentication methods](/reference/authentication). One of these is HTTPS Basic Authentication, which is recommended by the OpenRosa specification. However, because [we do not follow the try/retry pattern](/reference/authentication/https-basic-authentication/using-basic-authentication) required by the OpenRosa and the RFC specification, ODK Central is _not compliant_ with this component. Our recommendation generally is to use [Field Key Authentication](/reference/authentication/field-key-authentication) when submitting data from survey clients.
5. [**Form Discovery (Listing) API**](https://bitbucket.org/javarosa/javarosa/wiki/FormListAPI), which returns a listing of Forms available for survey clients to download and submit to. At this time, ODK Central is _partially compliant_ with this component: the server will return a correctly formatted `formList` response, but it does not currently handle the optional filter parameters.

In practical usage, ODK survey clients like Collect will interact with Central in three places:

* The OpenRosa Form Listing API, [documented below](/reference/openrosa-endpoints/openrosa-form-listing-api), lists the Forms the client can retrieve.
* The [Form XML download](/reference/forms-and-submissions/'-individual-form/retrieving-form-xml) endpoint, a part of the standard REST API for Forms, is linked in the Form Listing response and allows clients to then download the ODK XForms XML for each form.
* The OpenRosa Submission API, [documented below](/reference/openrosa-endpoints/openrosa-form-submission-api), allows survey clients to submit new Submissions to any Form.

The Form Listing and Submission APIs are partitioned by Project, and their URLs are nested under the Project in question as a result. When you List or Submit, you will only be able to get forms from and submit submissions to that particular Project at a time.

Where the **HTTP Request API** OpenRosa standards specification requires two headers for any request, Central requires only one:

* `X-OpenRosa-Version` **must** be set to exactly `1.0` or the request will be rejected.
* But Central does not require a `Date` header field. You may set it if you wish, but it will have no effect on Central.

## OpenRosa Form Listing API [GET /v1/projects/{projectId}/formList]

This is the mostly standards-compliant implementation of the [OpenRosa Form Discovery (Listing) API](https://bitbucket.org/javarosa/javarosa/wiki/FormListAPI). We will not attempt to redocument the standard here.

The following aspects of the standard are _not_ supported by ODK Central:

* The `deviceID` may be provided with the request, but it will be ignored.
* The `Accept-Language` header may be provided with the request, but it will be ignored.
* The `?formID=` querystring parameter is not supported and will be ignored.
* The `?verbose` querystring parameter is not supported and will be ignored.
* The `?listAllVersions` querystring is not supported and will be ignored. Central does not yet support multiple active versions of the same Form.
* No `<xforms-group/>` will ever be provided, as Central does not yet support this feature.

By default, the given `<name/>` in the Form Listing response is the friendly name associated with the `Form` (`<title>` in the XML and `name` on the API resource). If no such value can be found, then the `xmlFormId` will be given as the `<name>` instead.

A `<manifestUrl/>` property will be given per `<xform>` if and only if that form is expected to have media or data file attachments associated with it, based on its XForms definition. It will appear even if no attachments have actually been uploaded to the server to fulfill those expectations.

If you haven't already, please take a look at the **HTTP Request API** notes above on the required OpenRosa headers.

+ Parameters
    + projectId: `7` (number, required) - The numeric ID of the Project

+ Request
    + Headers

            X-OpenRosa-Version: 1.0

+ Response 200 (text/xml)
    + Headers

            X-OpenRosa-Version: 1.0

    + Body

            <?xml version="1.0" encoding="UTF-8"?>
            <xforms xmlns="http://openrosa.org/xforms/xformsList">
              <xform>
                <formID>basic</formID>
                <name>basic</name>
                <version></version>
                <hash>md5:a64817a5688dd7c17563e32d4eb1cab2</hash>
                <downloadUrl>https://your.odk.server/v1/projects/7/forms/basic.xml</downloadUrl>
                <manifestUrl>https://your.odk.server/v1/projects/7/forms/basic/manifest</manifestUrl>
              </xform>
              <xform>
                <formID>simple</formID>
                <name>Simple</name>
                <version></version>
                <hash>md5:</hash>
                <downloadUrl>https://your.odk.server/v1/projects/7/forms/simple.xml</downloadUrl>
              </xform>
            </xforms>

+ Response 403 (text/xml)
    + Headers

            X-OpenRosa-Version: 1.0

    + Body

            <OpenRosaResponse xmlns="http://openrosa.org/http/response" items="0">
              <message nature="error">The authenticated actor does not have rights to perform that action.</message>
            </OpenRosaResponse>

## OpenRosa Form Submission API [POST /v1/projects/{projectId}/submission]

This is the fully standards-compliant implementation of the [OpenRosa Form Submission API](https://bitbucket.org/javarosa/javarosa/wiki/FormSubmissionAPI). We will not attempt to redocument the standard here.

Some additional things to understand when using this API:

* ODK Central will always provide an `X-OpenRosa-Accept-Content-Length` of 100 megabytes. In reality, this number depends on how the server has been deployed. The default Docker-based installation, for example, is limited to 100MB at the nginx layer.
* The `xml_submission_file` may have a Content Type of either `text/xml` _or_ `application/xml`.
* Central supports the `HEAD` request preflighting recommended by the specification, but does not require it. Because our supported authentication methods do not follow the try/retry pattern, only preflight your request if you are concerned about the other issues listed in the standards document, like proxies.
* As stated in the standards document, it is possible to submit multimedia attachments with the `Submission` across multiple `POST` requests to this API. _However_, we impose the additional restriction that the Submission XML (`xml_submission_file`) _may not change_ between requests. If Central sees a Submission with an `instanceId` it already knows about but the XML has changed in any way, it will respond with a `409 Conflict` error and reject the submission.
* Central will never return a `202` in any response from this API.
* If you haven't already, please take a look at the **HTTP Request API** notes above on the required OpenRosa headers.

+ Parameters
    + projectId: `7` (number, required) - The numeric ID of the Project

+ Request (multipart/form-data; boundary=28b9211a964e44a3b327c3c51a0dbd32)
    + Headers

            X-OpenRosa-Version: 1.0

    + Body

            --28b9211a964e44a3b327c3c51a0dbd32
            Content-Disposition: form-data; name="xml_submission_file"; filename="submission.xml"
            Content-Type: application/xml

            <data id="simple"><meta><instanceID>uuid:85cb9aff-005e-4edd-9739-dc9c1a829c44</instanceID></meta><name>Alice</name><age>30</age></data>

            --28b9211a964e44a3b327c3c51a0dbd32--

+ Response 201 (text/xml)
    + Headers

            X-OpenRosa-Version: 1.0

    + Body

            <OpenRosaResponse xmlns="http://openrosa.org/http/response" items="0">
              <message nature="">full submission upload was successful!</message>
            </OpenRosaResponse>

+ Response 400 (text/xml)
    This is one of several possible `400` failures the API might return:

    + Headers

            X-OpenRosa-Version: 1.0

    + Body

            <OpenRosaResponse xmlns="http://openrosa.org/http/response" items="0">
              <message nature="error">A resource already exists with a attachment file name of attachment1.jpg.</message>
            </OpenRosaResponse>

+ Response 403 (text/xml)
    + Headers

            X-OpenRosa-Version: 1.0

    + Body

            <OpenRosaResponse xmlns="http://openrosa.org/http/response" items="0">
              <message nature="error">The authenticated actor does not have rights to perform that action.</message>
            </OpenRosaResponse>

+ Response 409 (text/xml)
    + Headers

            X-OpenRosa-Version: 1.0

    + Body

            <OpenRosaResponse xmlns="http://openrosa.org/http/response" items="0">
              <message nature="error">A submission already exists with this ID, but with different XML. Resubmissions to attach additional multimedia must resubmit an identical xml_submission_file.</message>
            </OpenRosaResponse>

## OpenRosa Form Manifest API [GET /v1/projects/{projectId}/forms/{xmlFormId}/manifest]

_(introduced: version 0.2.0)_

This is the fully standards-compliant implementation of the [OpenRosa Form Manifest API](https://bitbucket.org/javarosa/javarosa/wiki/FormListAPI#!the-manifest-document). We will not attempt to redocument the standard here.

A Manifest document is available at this resource path for any form in the system. However:

* A link to this document will not be given in the [Form Listing API](/reference/openrosa-endpoints/openrosa-form-listing-api) unless we expect the form to have media or data file attachments based on the XForms definition of the form.
* The Manifest will only output information for files the server actually has in its possession. Any missing expected files will be omitted, as we cannot provide a `hash` or `downloadUrl` for them.

+ Parameters
    + projectId: `7` (number, required) - The numeric ID of the Project

+ Request
    + Headers

            X-OpenRosa-Version: 1.0

+ Response 200 (text/xml)
    + Headers

            X-OpenRosa-Version: 1.0

    + Body

            <?xml version="1.0" encoding="UTF-8"?>
            <manifest xmlns="http://openrosa.org/xforms/xformsManifest">
              <mediaFile>
                <filename>question1.jpg</filename>
                <hash>md5:a64817a5688dd7c17563e32d4eb1cab2</hash>
                <downloadUrl>https://your.odk.server/v1/projects/7/forms/basic/attachments/question1.jpg</downloadUrl>
              </mediaFile>
              <mediaFile>
                <filename>question2.jpg</filename>
                <hash>md5:a6fdc426037143cf71cced68e2532e3c</hash>
                <downloadUrl>https://your.odk.server/v1/projects/7/forms/basic/attachments/question2.jpg</downloadUrl>
              </mediaFile>
            </manifest>

+ Response 403 (text/xml)
    + Headers

            X-OpenRosa-Version: 1.0

    + Body

            <OpenRosaResponse xmlns="http://openrosa.org/http/response" items="0">
              <message nature="error">The authenticated actor does not have rights to perform that action.</message>
            </OpenRosaResponse>

# Group OData Endpoints

[OData](http://www.odata.org/) is an emerging standard for the formal description and transfer of data between web services. In its most ambitious form, it aims to be _the_ standard way for any REST or REST-like API to enable interoperability with other services, not unlike the API Blueprint format this document is written in. However, in practical usage today it is primarily a way for data-centric web services to describe and transfer their data to various clients for deeper analysis or presentation.

ODK Central implements the [4.0 Minimal Conformance level](http://docs.oasis-open.org/odata/odata/v4.01/cs01/part1-protocol/odata-v4.01-cs01-part1-protocol.html#_Toc505771292) of the specification. Our goal is to enable data analysis and reporting through powerful third-party tools, sending them the data over OData, rather than attempt to create our own analysis tools. Today, our implementation primarily targets [Microsoft Power BI](https://docs.microsoft.com/en-us/power-bi/desktop-connect-odata) and [Tableau](https://onlinehelp.tableau.com/current/pro/desktop/en-us/examples_odata.html), two tools with reasonably robust free offerings that provide versatile analysis and visualization of data.

While OData itself supports data of any sort of structure, Power BI and Tableau both think in terms of relational tables. This presents an interesting challenge for representing ODK's `group` and `repeat` structures in OData. Our current solution is to treat every `repeat` in the `Form` definition as its own relational table, and we invent stable join IDs to relate the tables together.

In general, the OData standard protocol consists of three API endpoints:

* The **Service Document** describes the available resources in the service. We provide one of these for every `Form` in the system. In our case, these are the tables we derive from the `repeat`s in the given Form. The root table is always named `Submissions`.
* The **Metadata Document** is a formal XML-based EDMX schema description of every data object we might return. It is linked in every OData response.
* The actual data documents, linked from the Service Document, are a simple JSON representation of the submission data, conforming to the schema we describe in our Metadata Document.

As our focus is on the bulk-export of data from ODK Central so that more advanced analysis tools can handle the data themselves, we do not support most of the features at the Intermediate and above conformance levels, like `$sort` or `$filter`.

## OData Form Service [/v1/projects/{projectId}/forms/{xmlFormId}.svc]

ODK Central presents one OData service for every `Form` it knows about. Each service might have multiple tables related to that Form. To access the OData service, simply add `.svc` to the resource URL for the given Form.

+ Parameters
    + projectId: `7` (number, required) - The numeric ID of the Project

    + `xmlFormId`: `simple` (string, required) - The `xmlFormId` of the `Form` whose OData service you wish to access.

### Service Document [GET]

The Service Document is essentially the index of all available documents. From this document, you will find links to all other available information in this OData service. In particular, you will find the Metadata Document, as well as a data document for each table defined by the `Form`.

You will always find a link to the `/Submissions` subpath, which is the data document that represents the "root" table, the data from each `Submission` that is not nested within any `repeat`s.

This document is available only in JSON format.

+ Response 200 (application/json; charset=utf-8; odata.metadata=minimal)
    + Body

            {
                "@odata.context": "https://your.odk.server/v1/projects/7/forms/sample.svc/$metadata",
                "value": [
                    {
                        "kind": "EntitySet",
                        "name": "Submissions",
                        "url": "Submissions"
                    },
                    {
                        "kind": "EntitySet",
                        "name": "Submissions.children.child",
                        "url": "Submissions.children.child"
                    }
                ]
            }

+ Response 403 (application/json)
    + Attributes (Error 403)

+ Response 406 (application/json)
    + Attributes (Error 406)

### Metadata Document [GET /v1/projects/{projectId}/forms/{xmlFormId}.svc/$metadata]

The Metadata Document describes, in [EDMX CSDL](http://docs.oasis-open.org/odata/odata-csdl-xml/v4.01/odata-csdl-xml-v4.01.html), the schema of all the data you can retrieve from the OData Form Service in question (essentially, this is the XForms form schema translated into the OData format). EDMX/CSDL is very similar in concept to UML: there are objects, they have properties, and some of those properties are relationships to other objects.

If you are writing a tool to analyze your own data, whose schema you already know and understand, there is very little reason to touch this endpoint at all. You can likely skip ahead to the data documents themselves and work directly with the simple JSON output returned by those endpoints. This endpoint is more useful for authors of tools which seek to generically work with arbitrary data whose schemas they cannot know in advance.

In general, the way we model the XForms schema in OData terms is to represent `group`s as `ComplexType`s, and `repeat`s as `EntityType`s. In the world of OData, the primary difference between these two types is that Entity Types require Primary Keys, while Complex Types do not. This fits well with the way XForms surveys tend to be structured.

Due to a limitation in Power BI, we do not take the extra step of advertising the actual relationships between tables (the point at which a `repeat` connects the parent data to the repeated subtable). Normally, this would be done with a `NavigationProperty`. However, while the OData specification allows Navigation Properties to exist on Complex Types, Power BI only allows them on Entity Types, as it makes certain assumptions about how Primary Keys must be structured to relate the two tables together. This is a problem for us, because it is a common idiom in ODK XForms design to place `repeat`s inside of `group`s. When Power BI resolves this issue, we will be able to formally represent the joins between these tables.

This implementation of the OData standard includes a set of Annotations describing the supported features of the service in the form of the [Capabilities Vocabulary](https://github.com/oasis-tcs/odata-vocabularies/blob/master/vocabularies/Org.OData.Capabilities.V1.md). In general, however, you can assume that the server supports the Minimal Conformance level and nothing beyond.

While the latest 4.01 OData specification adds a new JSON EDMX CSDL format, most servers and clients do not yet support that format, and so for this release of ODK Central only the older XML EDMX CSDL format is available.

+ Parameters
    + `xmlFormId`: `simple` (string, required) - The `xmlFormId` of the `Form` whose OData service you wish to access.

+ Response 200 (application/xml)
      + Body

              <?xml version="1.0" encoding="UTF-8"?>
              <edmx:Edmx xmlns:edmx="http://docs.oasis-open.org/odata/ns/edmx" Version="4.0">
                <edmx:DataServices>
                  <Schema xmlns="http://docs.oasis-open.org/odata/ns/edm" Namespace="org.opendatakit.user.simple">
                    <EntityType Name="Submissions">
                      <Key><PropertyRef Name="__id"/></Key>
                      <Property Name="__id" Type="Edm.String"/>
                      <Property Name="meta" Type="org.opendatakit.user.simple.meta"/>
                      <Property Name="name" Type="Edm.String"/>
                      <Property Name="age" Type="Edm.Int64"/>
                    </EntityType>
                    <ComplexType Name="meta">
                      <Property Name="instanceID" Type="Edm.String"/>
                    </ComplexType>
                    <EntityContainer Name="simple">
                      <EntitySet Name="Submissions" EntityType="org.opendatakit.user.simple.Submissions">
                        <Annotation Term="Org.OData.Capabilities.V1.ConformanceLevel" EnumMember="Org.OData.Capabilities.V1.ConformanceLevelType/Minimal"/>
                        <Annotation Term="Org.OData.Capabilities.V1.BatchSupported" Bool="false"/>
                        <Annotation Term="Org.OData.Capabilities.V1.CountRestrictions">
                          <Record><PropertyValue Property="Countable" Bool="true"/></Record>
                        </Annotation>
                        <Annotation Term="Org.OData.Capabilities.V1.FilterFunctions">
                          <Record>
                            <PropertyValue Property="NonCountableProperties">
                              <Collection>
                                <String>eq</String>
                              </Collection>
                            </PropertyValue>
                          </Record>
                        </Annotation>
                        <Annotation Term="Org.OData.Capabilities.V1.FilterFunctions">
                          <Record>
                            <PropertyValue Property="Filterable" Bool="true"/>
                            <PropertyValue Property="RequiresFilter" Bool="false"/>
                            <PropertyValue Property="NonFilterableProperties">
                              <Collection>
                                <PropertyPath>meta</PropertyPath>
                                <PropertyPath>name</PropertyPath>
                                <PropertyPath>age</PropertyPath>
                              </Collection>
                            </PropertyValue>
                          </Record>
                        </Annotation>
                        <Annotation Term="Org.OData.Capabilities.V1.SortRestrictions">
                          <Record><PropertyValue Property="Sortable" Bool="false"/></Record>
                        </Annotation>
                        <Annotation Term="Org.OData.Capabilities.V1.ExpandRestrictions">
                          <Record><PropertyValue Property="Expandable" Bool="false"/></Record>
                        </Annotation>
                      </EntitySet>
                    </EntityContainer>
                  </Schema>
                </edmx:DataServices>
              </edmx:Edmx>

+ Response 403 (application/json)
    + Attributes (Error 403)

+ Response 406 (application/json)
    + Attributes (Error 406)

### Data Document [GET /v1/projects/{projectId}/forms/{xmlFormId}.svc/{table}{?%24skip,%24top,%24count,%24wkt}]

The data documents are the straightforward JSON representation of each table of `Submission` data. They follow the [corresponding specification](http://docs.oasis-open.org/odata/odata-json-format/v4.01/odata-json-format-v4.01.html), but apart from the representation of geospatial data as GeoJSON rather than the ODK proprietary format, the output here should not be at all surprising. If you are looking for JSON output of Submission data, this is the best place to look.

The `$top` and `$skip` querystring parameters, specified by OData, apply `limit` and `offset` operations to the data, respectively. The `$count` parameter, also an OData standard, will annotate the response data with the total row count, regardless of the scoping requested by `$top` and `$skip`. While paging is possible through these parameters, it will not greatly improve the performance of exporting data. ODK Central prefers to bulk-export all of its data at once if possible.

In this release of Central, `$expand` is not yet supported. This will likely change in the future, once we can instate Navigation Properties.

The _nonstandard_ `$wkt` querystring parameter may be set to `true` to request that geospatial data is returned as a [Well-Known Text (WKT) string](https://en.wikipedia.org/wiki/Well-known_text) rather than a GeoJSON structure. This exists primarily to support Tableau, which cannot yet read GeoJSON, but you may find it useful as well depending on your mapping software. **Please note** that both GeoJSON and WKT follow a `(lon, lat, alt)` coördinate ordering rather than the ODK-proprietary `lat lon alt`. This is so that the values map neatly to `(x, y, z)`. GPS accuracy information is not a part of either standards specification, and so is presently omitted from OData output entirely. GeoJSON support may come in a future version.

As the vast majority of clients only support the JSON OData format, that is the only format ODK Central offers.

+ Parameters
    + `xmlFormId`: `simple` (string, required) - The `xmlFormId` of the `Form` whose OData service you wish to access.
    + `table`: `Submissions` (string, required) - The name of the table to be returned. These names can be found in the output of the [Service Document](/reference/odata-endpoints/odata-form-service/service-document).
    + `%24skip`: `10` (number, optional) - If supplied, the first `$skip` rows will be omitted from the results.
    + `%24top`: `5` (number, optional) - If supplied, only up to `$top` rows will be returned in the results.
    + `%24count`: `true` (boolean, optional) - If set to `true`, an `@odata.count` property will be added to the result indicating the total number of rows, ignoring the above paging parameters.
    + `%24wkt`: `true` (boolean, optional) - If set to `true`, geospatial data will be returned as Well-Known Text (WKT) strings rather than GeoJSON structures.

+ Response 200 (application/json)
    + Body

            {
                "@odata.context": "https://your.odk.server/v1/projects/7/forms/simple.svc/$metadata#Submissions",
                "value": [
                    {
                        "__id": "uuid:85cb9aff-005e-4edd-9739-dc9c1a829c44",
                        "age": 25,
                        "meta": {
                            "instanceID": "uuid:85cb9aff-005e-4edd-9739-dc9c1a829c44"
                        },
                        "name": "Bob"
                    },
                    {
                        "__id": "uuid:297000fd-8eb2-4232-8863-d25f82521b87",
                        "age": 30,
                        "meta": {
                            "instanceID": "uuid:297000fd-8eb2-4232-8863-d25f82521b87"
                        },
                        "name": "Alice"
                    }
                ]
            }

+ Response 403 (application/json)
    + Attributes (Error 403)

+ Response 406 (application/json)
    + Attributes (Error 406)

+ Response 501 (application/json)
    + Attributes (Error 501)

# Group System Configuration

Currently, the only system configuration present in ODK Central is the configuration of periodic backups. That API is described here.

## Backups Configuration [/v1/config/backups]

This resource does not conform to REST standards, as it has a multi-step initialization process and is essentially a singleton object: only one backup may be configured at a time.

### Getting the current configuration [GET]

If configured, this endpoint will return a combination of the present backups configuration type, along with the `Audit` log entry of the most recent backup attempt, if one exists. For security reasons, none of the actual internal authentication and encryption information is returned.

If no backups are configured, this endpoint will return a `404`.

+ Response 200 (application/json)
    + Body

            {
                "config": {
                    "type": "google"
                },
                "latest": {
                    "acteeId": null,
                    "action": "backup",
                    "actorId": null,
                    "details": {
                        "success": true
                    },
                    "loggedAt": "2018-03-21T03:47:03.504Z"
                }
            }

+ Response 403 (application/json)
    + Attributes (Error 403)

+ Response 404 (application/json)
    + Attributes (Error 404)

### Terminating the current configuration [DELETE]

Deleting the backups configuration will permanently remove it, stopping it from running as scheduled.

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

### Initiating a new backup configuration [POST /v1/config/backups/initiate]

This is the first of two steps required to initialize a new backup configuration.

While the administrative interface packaged with ODK Central requires a backup to be terminated before a new one may be set up, the API has no such limitation. Should the initialization process succeed (with a `POST /v1/config/backups/verify` as [documented below](/reference/system-configuration/backups-configuration/completing-a-new-backup-configuration)), the existing backup will be overwritten with the new configuration.

All ODK Central backups are encrypted before they are sent to Google Drive for storage. If a `passphrase` is not provided with this request, encryption will still occur with a `""` empty string passphrase.

To complete the backup configuration, it is necessary to use the Google OAuth URL returned by this endpoint and have a human interactively undergo the Google OAuth process to authorize ODK Central to send files to their account. Because of the way we have set the OAuth structure up, it is not necessary for each installation of the server to provision its own Google Developer key.

+ Request (application/json)
    + Attributes
        + passphrase: `super.secret` (string, optional) - The passphrase to use in encrypting the backup data. If no passphrase is provided, backups are still encrypted but with a `""` empty string passphrase.

+ Response 200 (application/json)
    + Attributes
        + token: `DS9aOoudH3kGw3a$tkCf9SGa3gvM41sbuZ$2IpVTrc!OBbKjrB03JPXA0Csn1lVF` (string, required) - The Session Bearer Token to be used in authenticating the follow-up `/verify` request.
        + url: `https://accounts.google.com/o/oauth2/v2/auth?access_type=offline&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fdrive.file&response_type=code&client_id=660095633112-h7bhsjenhp1agd0c4v3cmqk6bccgkdu0.apps.googleusercontent.com&redirect_uri=urn%3Aietf%3Awg%3Aoauth%3A2.0%3Aoob` (string, required) - The Google OAuth URL to send the human user to in order to allow access to their Google Drive account for storing backups.

+ Response 403 (application/json)
    + Attributes (Error 403)

### Completing a new backup configuration [POST /v1/config/backups/verify]

This is the second of two steps required to initialize a new backup configuration. As noted in step 1 above, if a backup already exists and this request succeeds, it will be overwritten with the new configuration.

This endpoint has two requirements, both relating to [step 1](/reference/system-configuration/backups-configuration/initiating-a-new-backup-configuration):

* It _must_ be called via Session Bearer Token auth (`Authorization: Bearer {token}`), with the token provided in the response to step 1. Attempting to authenticate as any other `Actor` will fail.
* It needs to be provided with the verification `code` the user receives at the end of their Google OAuth process.

If these two things are present and correct, and Google can be reached to verify the code, the new backups will be configured.

+ Request (application/json)
    + Attributes
        + code: `4/AACdfD3bBxUAI-zlkLodGmUh_IC4zT6-j8EJ7jr7IFF7Nm4RH_S7RHs` (string, required) - The code the user receives at the end of the OAuth authorization process. Typically, it begins with the text `4/AA`.

+ Response 200 (application/json)
    + Attributes (Success)

+ Response 403 (application/json)
    + Attributes (Error 403)

# Data Structures

These are in alphabetic order, with the exception that the `Extended` versions of each structure, where applicable, is listed immediately following the standard version.

## Actor (object)
+ createdAt: `2018-04-18T23:19:14.802Z` (string, required) - ISO date format
+ displayName: `My Display Name` (string, required) - All `Actor`s, regardless of type, have a display name
+ id: `115` (number, required)
+ meta: `{ "json": "data" }` (string, optional) - Sometimes additional information is tracked in this JSON object. Modify with care.
+ updatedAt: `2018-04-18T23:42:11.406Z` (string, optional) - ISO date format

## Error 400 (object)
+ code: `400` (string, required)
+ details (object, optional) - a subobject that contains programmatically readable details about this error
+ message: `Could not parse the given data (2 chars) as json.` (string)

## Error 401 (object)
+ code: `401.2` (string, required)
+ message: `Could not authenticate with the provided credentials.` (string)

## Error 403 (object)
+ code: `403.1` (string, required)
+ message: `The authenticated actor does not have rights to perform that action.` (string)

## Error 404 (object)
+ code: `404.1` (string, required)
+ message: `Could not find the resource you were looking for.` (string)

## Error 406 (object)
+ code: `406.1` (string, required)
+ message: `Requested format not acceptable; this resource allows: (application/json, json).` (string)

## Error 501 (object)
+ code: `501.1` (string, required)
+ message: `The requested feature $unsupported is not supported by this server.` (string)

## Field Key (Actor)
+ createdBy: `42` (number, required) - The ID of the `Actor` that created this `Field Key`.
+ token: `d1!E2GVHgpr4h9bpxxtqUJ7EVJ1Q$Dusm2RBXg8XyVJMCBCbvyE8cGacxUx3bcUT` (string, optional) - If present, this is the Token that can be used to authenticate a request as this `Field Key`. If not present, this `Field Key`'s access has been revoked.

## Form (object)
+ xmlFormId: `simple` (string, required) - The `id` of this form as given in its XForms XML definition
+ name: `Simple` (string, optional) - The friendly name of this form. It is given by the `<title>` in the XForms XML definition.
+ version: `2.1` (string, optional) - The `version` of this form as given in its XForms XML definition. Empty string and `null` are treated equally as a single version.
+ hash: `51a93eab3a1974dbffc4c7913fa5a16a` (string, required) - An MD5 sum automatically computed based on the XForms XML definition. This is required for OpenRosa compliance.
+ state (Form State, required) - The present lifecycle status of this form. Controls whether it is available for download on survey clients or accepts new submissions.
+ createdAt: `2018-01-19T23:58:03.395Z` (string, required) - ISO date format
+ updatedAt: `2018-03-21T12:45:02.312Z` (string, optional) - ISO date format

## Extended Form (Form)
+ submissions: `10` (number, required) - The number of `Submission`s that have been submitted to this `Form`.
+ lastSubmission: `2018-04-18T03:04:51.695Z` (string, optional) - ISO date format. The timestamp of the most recent submission, if any.
+ xml: `…` (string, required) - The XForms XML that defines this form.

## Form Attachment (object)
+ name: `myfile.mp3` (string, required) - The name of the file as specified in the XForm.
+ type: (Form Attachment Type, required) - The expected type of file as specified in the XForm.
+ exists: `true` (boolean, required) - Whether the server has the file or not.

## Extended Form Attachment (Form Attachment)
+ updatedAt: `2018-03-21T12:45:02.312Z` (string, optional) - ISO date format. The last time this file's binary content was set (POST) or cleared (DELETE).

## Form Attachment Type (enum)
+ image (string) - An image (jpg, etc) file is expected.
+ audio (string) - An audio (mp3, etc) file is expected.
+ video (string) - A video (mp4, etc) file is expected.
+ file (string) - A data file of some kind (usually csv) is expected.

## Form State (enum)
+ open (string) - _(Default)_ This form is available for download and accepts submissions.
+ closing (string) - This form is _not_ available for download but still accepts submissions.
+ closed (string) - This form is _not_ available for download, and it does _not_ accept submissions.

## Extended Field Key (Field Key)
+ createdBy (Actor, required) - The full details about the `Actor` that created this `Field Key`.
+ lastUsed: `2018-04-14T08:34:21.633Z` (string, optional) - ISO date format. The last time this `Field Key` was used to authenticate a request.

## Project (object)
  + id: `1` (number) - The numerical ID of the Project.
  + name: `Default Project` (string, required) - The name of the Project.

## Extended Project (Project)
  + forms: `7` (number) - The number of forms within this Project.
  + lastSubmission: `2018-04-18T03:04:51.695Z` (string, optional) - ISO date format. The timestamp of the most recent submission to any form in this project, if any.

## User (Actor)
+ email: `my.email.address@opendatakit.org` (string, required) - Only `User`s have email addresses associated with them

## Session (object)
+ createdAt: `2018-04-18T03:04:51.695Z` (string, required) - ISO date format
+ expiresAt: `2018-04-19T03:04:51.695Z` (string, required) - ISO date format
+ token: `lSpAIeksRu1CNZs7!qjAot2T17dPzkrw9B4iTtpj7OoIJBmXvnHM8z8Ka4QPEjR7` (string, required) - The bearer token associated with the session. It consists only of URL-safe characters, so it should never need any escaping.

## Submission (object)
+ instanceId: `uuid:85cb9aff-005e-4edd-9739-dc9c1a829c44` (string, required) - The `instanceId` of the `Submission`, given by the Submission XML.
+ submitter: `23` (number, required) - The ID of the `Actor` (`Field Key` or `User`) that submitted this `Submission`.
+ createdAt: `2018-01-19T23:58:03.395Z` (string, required) - ISO date format
+ updatedAt: `2018-03-21T12:45:02.312Z` (string, optional) - ISO date format

## Extended Submission (Submission)
+ submitter (Actor, required) - The full details of the `Actor` that submitted this `Submission`.
+ xml: `…` (string, required) - The actual `Submission` XML.

## Submission Attachment (object)
+ name: `myfile.mp3` (string, required) - The name of the file as specified in the Submission XML.
+ exists: `true` (boolean, required) - Whether the server has the file or not.

## Success (object)
+ success: `true` (boolean, required)

