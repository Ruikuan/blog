# Why using base64 encoding json transfer large file is evil

REST is a resource-centric design, focusing on resource identifiers and corresponding methods. Most of our platform’s APIs don’t follow this style, they are just some HTTP/web APIs. 

Apis mostly adopt JSON as the JSON payload does fit very well when API’s arguments are small, it’s clear and human-readable. But not necessarily always JSON when the payload is large.

Strings in dotnet are immutable and can not be reused, they are allocated then thrown away immediately. Large strings (>85KB, when image > 56KB cause the base64 overhead) will be allocated into LOH directly, many LOH allocations will cause full GC frequently, that will hurt the service’s performance and reliability badly.

When talking about binary protocol, it often means combining operation + arguments and encoding them into an opaque, unreadable binary format. Uploading a blob to the server is not likely a binary protocol, the blob is the file content itself, without any additional encoding. Actually, encoding the blob into a opaque base64 string representation then transferring it with a JSON doesn’t help the readability and understanding.

In the scenario of uploading relatively large files, it is not uncommon to upload the raw blob directly, AWS S3 put object does it this way, [Azure face detect API](https://westus.dev.cognitive.microsoft.com/docs/services/563879b61984550e40cbbe8d/operations/563879b61984550f30395236 ) also supports it. They all call these APIs “REST API“… 

So uploading the blob maybe can be a feasible option. Uploading the blob in javascript/browser via fetch is straightforward, and should not add too much complexity.
