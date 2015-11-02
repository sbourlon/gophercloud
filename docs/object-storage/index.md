---
layout: page
title: Getting Started with Object Storage v1
---

* [Setup](#setup)
* [Containers](#containers)
	* [Create container](#create-container)
	* [List containers](#list-containers)
	* [Metadata](#container-metadata)
	* [Delete container](#delete-container)
* [Objects](#objects)
	* [Upload objects](#upload-object)
	* [List objects](#list-objects)
	* [Copying objects](#copy-object)
	* [Download objects](#download-object)
	* [Metadata](#object-metadata)
* [Account](#account)
	* [View metadata](#view-account-metadata)
	* [Update metadata](#update-account-metadata)
* [Providers](#providers)
	* [Rackspace](#rackspace)

## <a name="setup"></a>Setup

In order to interact with OpenStack APIs, you must first pass in your auth
credentials to a `Provider` struct. Once you have this, you then retrieve
whichever service struct you're interested in - so in our case, we invoke the
`NewObjectStorageV1` method:

{% highlight go %}
authOpts, err := openstack.AuthOptionsFromEnv()

provider, err := openstack.AuthenticatedClient(authOpts)

client, err := openstack.NewObjectStorageV1(provider, gophercloud.EndpointOpts{
	Region: "RegionOne",
})
{% endhighlight %}

If you're unsure about how to retrieve credentials, please read our [introductory
guide](/docs) which outlines the steps you need to take.

## <a name="containers"></a>Containers

A container is a storage compartment that provides a way for you to organize
your objects. It is analogous to a Linux directory or Windows folder, with the
exception that you cannot nest containers in other containers like a filesystem.

### <a name="create-container"></a>Create a new container

{% highlight go %}
import "github.com/rackspace/gophercloud/openstack/objectstorage/v1/containers"

// We have the option of passing in configuration options for our new container
opts := containers.CreateOpts{
	ContainerSyncTo: "backup_container",
	Metadata:        map[string]string{"author": "emily dickinson"},
}

res := containers.Create(client, "container_name", opts)

// If we want to extract information out from the response headers, we can.
// The first return value will be http.Header (alias of map[string][]string).
headers, err := res.ExtractHeader()
{% endhighlight %}

### <a name="list-containers"></a>List containers

{% highlight go %}
import "github.com/rackspace/gophercloud/pagination"

// We have the option of filtering containers by their attributes
opts := &containers.ListOpts{Full: true, Prefix: "backup_"}

// Retrieve a pager (i.e. a paginated collection)
pager := containers.List(client, opts)

// Define an anonymous function to be executed on each page's iteration
err := pager.EachPage(func(page pagination.Page) (bool, error) {

	// Get a slice of containers.Container structs
	containerList, err := containers.ExtractInfo(page)
	for _, c := range containerList {
		// ...
	}

	// Get a slice of strings, i.e. container names
	containerNames, err := containers.ExtractNames(page)
	for _, n := range containerNames {
		// ...
	}

	return true, nil
})
{% endhighlight %}

### <a name="container-metadata"></a>View and modify container metadata

To retrieve a container's metadata:

{% highlight go %}
metadata, err := containers.Get(client, "container_name").ExtractMetadata()

// Iterate over the map[string]string
for key, val := range metadata {
	// ...
}
{% endhighlight %}

To update a container's metadata:

{% highlight go %}
// We need to specify the new metadata. Keys that do not exist will be added,
// keys that already exist will be overriden. Keys that are not included in
// this struct will be deleted.
opts := &containers.UpdateOpts{
	Metadata: map[string]string{"new_key": "new_value"},
}

result := containers.Update(client, "container_name", opts)
{% endhighlight %}

### <a name="delete-container"></a>Delete an existing container

{% highlight go %}
response := containers.Delete(client, "container_name")

// Like most operations, we can extract headers values too
headers, err := response.ExtractHeader()
{% endhighlight %}

## <a name="objects"></a>Objects

An object stores data content, such as documents, images, and so on. Another way
to think about it is that it's like a traditional file on a local filesystem
but with lots of additional functionality. For example, you can store custom
metadata on an object, compress files, manage access with CORS and temporary
URLs, schedule deletions, and execute batch operations (like deleting 10,000
objects at a time).

### <a name="upload-object"></a>Upload objects

When uploading a new object, you need to the container name you're
uploading to, and the name of your new object. You also need to provide the
content of your object - and to do this, you need to use Golang's standard
[`io.ReadSeeker`](http://golang.org/pkg/io/#ReadSeeker) interface.

The first thing you need If you want to upload the contents of a local file:

{% highlight go %}
import "os"

content, err := os.Open("/path/to/file")
{% endhighlight %}

or to use a basic string:

{% highlight go %}
import "strings"

content := strings.NewReader("your string")
{% endhighlight %}

or to use a slice of bytes:

{% highlight go %}
import "bytes"

content := bytes.NewReader(bytes)
{% endhighlight %}

Once you have your content in the form of a reader, you can create your object:

{% highlight go %}
import "github.com/rackspace/gophercloud/openstack/objectstorage/v1/objects"

// You have the option of specifying additional configuration options.
opts := objects.CreateOpts{
	ContentDisposition: `attachment; filename="foo_bar.pdf"`,
	DeleteAfter: 3600,
}

// Now execute the upload
res := objects.Create(client, "container_name", "object_name", content, opts)

// We have the option of extracting the resulting headers from the response
headers, err := res.ExtractHeader()
{% endhighlight %}

### <a name="list-objects"></a>List objects

{% highlight go %}
import "github.com/rackspace/gophercloud/pagination"

// We have the option of filtering objects by their attributes
opts := &objects.ListOpts{Full: true}

// Retrieve a pager (i.e. a paginated collection)
pager := objects.List(client, "container_name", opts)

// Define an anonymous function to be executed on each page's iteration
err := pager.EachPage(func(page pagination.Page) (bool, error) {

	// Get a slice of objects.Object structs
	objectList, err := objects.ExtractInfo(page)
	for _, n := range objectNames {
		// ...
	}

	// Get a slice of strings, i.e. object names
	objectNames, err := containers.ExtractNames(page)
	for _, n := range objectNames {
		// ...
	}

	return true, nil
})
{% endhighlight %}

### <a name="copy-object"></a>Copy to new location

Let's say we want to copy `logs/wednesday_14th` to `backup/wednesday_14th_2014`.
This operation always creates a new object. If you use this operation against an
existing object, you replace the existing object and metadata rather than
modifying the object.

{% highlight go %}
// Define our options
opts := &objects.CopyOpts{Destination: "backup/wednesday_14th_2014"}

// Perform the copy
result := objects.Copy(client, "logs", "wednesday_14th", opts)

// Extract response headers
headers, err := result.ExtractHeader()
{% endhighlight %}

### <a name="download-object"></a>Download object

{% highlight go %}
// Configure options
opts := objects.DownloadOpts{IfUnmodifiedSince: "date"}

// Download everything into a DownloadResult struct
res := objects.Download(client, "container_name", "object_name", opts)

// Extract a slice of bytes
bytes, err := res.ExtractContent()

// Extract headers
header, err := res.ExtractHeader()
{% endhighlight %}

### <a name="object-metadata"></a>Retrieve and update metadata

{% highlight go %}
// We can specify additional options (to enable conditional requests for example)
opts := objects.DownloadOpts{IfMatch: "etag"}

// To perform the download operation
result := objects.Download(client, "container_name", "object_name", opts)

// To extract content out into a slice of bytes, we can use:
bytes, err := result.ExtractContent()
{% endhighlight %}

### <a name="delete-object"></a>Delete object

{% highlight go %}
result := objects.Delete(client, "container_name", "object_name")
{% endhighlight %}

## <a name="account"></a>Account

An account represents the very top-level namespace of the resource hierarchy -
containers belong to accounts, and objects belong to containers. Normally your
service provider creates your account and you then own and can control all the
resources in that account. The account defines a namespace for containers. In
the OpenStack environment, account is synonymous with a project or a tenant.

### <a name="view-account-metadata"></a>Retrieve metadata

{% highlight go %}
import "github.com/rackspace/gophercloud/openstack/objectstorage/v1/accounts"

// Get information from the API
res := accounts.Get(client, GetOpts{})

// Extract metadata out of it
metadata := res.ExtractMetadata()

for k, v := range metadata {
	// ...
}
{% endhighlight %}

### <a name="update-account-metadata"></a>Update metadata

{% highlight go %}
// Set new metadata
opts := accounts.UpdateOpts{
	Metadata: map[string]string{"foo": "bar"}
}

// Send to API
res := accounts.Update(client, opts)

// Extract metadata out of it
metadata := res.ExtractMetadata()

for k, v := range metadata {
	// ...
}
{% endhighlight %}

## <a name="providers"></a>Providers

### <a name="rackspace"></a>Rackspace

* [Quickstart for Cloud Files](https://developer.rackspace.com/docs/cloud-files/getting-started/?lang=go)
on the Rackspace Developer portal.