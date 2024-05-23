# wolfHSM Client Library

The client library API is the primary mechanism through which users will interact with wolfHSM. The full client API documentation can be found in (chapter x)[todo].


## API Return Codes

All client API functions return a wolfHSM error code indicating success or the type of failure. Some failures are critical errors, while others may simply indicate an action is required from the caller (e.g. `WH_ERROR_NOTREADY` in the case of a non-blocking operation). Many client APIs also propagate a server error code (and in some cases an additional status) to the caller, allowing for the case where the underlying request transaction succeeded but the server was unable to perform the operation. Examples of this include requesting an NVM object from the server that doesn't exist, attempting to add an object when NVM is full, or trying to use a cryptographic algorithm that the server is not configured to support. 

## Split Transaction Processing

Most client APIs are fully asynchronous and decomposed into split transactions, meaning there is a separate function for the operation request and response. The request function sends the request to the server and immediately returns without blocking. The response function polls the underlying transport for a response, processing it if it exists, and immediately returning if it has not yet arrived. This allows for the client to request long running operations from the server without wasting client CPU cycles. The following example shows an example asynchronous request and response invocation using the "echo" message:

```c
int rc;

/* send an echo request */
rc = wh_Client_EchoRequest(&clientCtx, sendLen, &sendBuffer);
if (rc != WH_ERROR_OK) {
    /* handle error */
}

/* do work... */

/* poll for a server response */
while ((rc = wh_Client_EchoResponse(client, &recv_len, recv_buffer)) == WH_ERROR_NOTREADY) {
    /* do work or yield */
}

if (rc != WH_ERROR_OK) {
    /* handle error */
}


```

## The Client Context

The client context structure (`whClientContext`) holds the runtime state of the client and represents the enpoint of the connection with the server. There is a one-to-one relationship between client and server contexts, meaning an application that interacts with multiple servers will need multiple client contexts - one for each server. Each client API function takes a client context as an argument, indicating which server connection the operations will correspond to. If familiar with wolfSSL, the client context structure is analogous to the `WOLFSSL` connection context structure.

### Initializing the client context 

Before using any client APIs on a client context, the structure must be configured and initialized using the `whClientConfig` configuration structure and the `wh_Client_Init()` function.

The client configuration structure holds the communication layer configuration (`whCommClientConfig`) that will be used to configure and initialize the context for the server communication. The `whCommClientConfig` structure binds an actual transport implementation (either built-in or custom) to the abstract comms interface for the client to use.

The general steps to configure a client are:

1. Allocate and initialize a transport configuration structure, context, and callback implementation for the desired transport
2. Allocate and comm client configuration structure and bind it to the transport configuration from step 1 so it can be used by the client
3. Allocate and initialize a client configuration structure using the comm client configuration in step 2
4. Allocate a client context structure
5. Initialize the client with the client configuration by calling `wh_Client_Init()`
6. Use the client APIs to interact with the server

Here is a bare-minimum example of configuring a client application to use the built-in shared memory transpor

```c
#include <string.h> /* for memcmp() */
#include "wolfhsm/client.h"  /* Client API (includes comm config) */
#include "wolfhsm/wh_transport_mem.h" /* transport implementation */

/* Step 1: Allocate and initialize the shared memory transport configuration */
/* Shared memory transport configuration */
static whTransportMemConfig transportMemCfg = { /* shared memory config */ };
/* Shared memory transport context (state) */
whTransportMemClientContext transportMemClientCtx = {0};
/* Callback structure that binds the abstract comm transport interface to
 * our concrete implementation */
whTransportClientCb transportMemClientCb = {WH_TRANSPORT_MEM_CLIENT_CB};

/* Step 2: Allocate client comm configuration and bind to the transport */
/* Configure the client comms to use the selected transport configuration */
whCommClientConfig commClientCfg[1] = {{
             .transport_cb      = transportMemClientCb,
             .transport_context = (void*)transportMemClientCtx,
             .transport_config  = (void*)transportMemCfg,
             .client_id         = 123, /* unique client identifier */
}};

/* Step 3: Allocate and initialize the client configuration */
whClientConfig clientCfg= {
   .comm = commClientCfg,
};

/* Step 4: Allocate the client context */
whClientContext clientCtx = {0};

/* Step 5: Initialize the client with the provided configuration */
wh_Client_Init(&clientCtx, &clientCfg);
```

The client context is now intialized and can be used with the client library API functions in order to do work. Here is an example of sending an echo request to the server:

```c
/* Step 6: Use the client APIs to interact with the server */

/* Buffers to hold sent and received data */
char recvBuffer[WH_COMM_DATA_LEN] = {0};
char sendBuffer[WH_COMM_DATA_LEN] = {0};

uint16_t sendLen = snprintf(&sendBuffer,
                            sizeof(sendBuffer),
                            "Hello World!\n");
uint16_t recvLen = 0;

/* Send an echo request and block on receiving a response */
wh_Client_Echo(client, sendLen, &sendBuffer, &recvLen, &recvBuffer);

if ((recvLen != sendLen ) ||
    (0 != memcmp(sendBuffer, recvBuffer, sendLen))) {
    /* Error, we weren't echoed back what we sent */
}
```

While there are indeed a large number of nested configurations and structures to set up, designing wolfHSM this way allowed for different transport implementations to be swapped in and out easily without changing the client code. For example, in order to switch from the shared memory transport to a TCP transport, only the transport configuration and callback structures need to be changed, and the rest of the client code remains the same (everything after step 2 in the sequence above).

```c
#include <string.h> /* for memcmp() */
#include "wolfhsm/client.h"  /* Client API (includes comm config) */
#include "port/posix_transport_tcp.h" /* transport implementation */

/* Step 1: Allocate and initialize the posix TCP transport configuration */
/* Client configuration/contexts */
whTransportClientCb posixTransportTcpCb = {PTT_CLIENT_CB};
posixTransportTcpClientContext posixTranportTcpCtx = {0};
posixTransportTcpConfig posixTransportTcpCfg = {
    /* IP and port configuration */
};

/* Step 2: Allocate client comm configuration and bind to the transport */
/* Configure the client comms to use the selected transport configuration */
whCommClientConfig commClientCfg = {{
             .transport_cb      = posixTransportTcpCb,
             .transport_context = (void*)posixTransportTcpCtx,
             .transport_config  = (void*)posixTransportTcpCfg,
             .client_id         = 123, /* unique client identifier */
}};

/* Subsequent steps remain the same... */
```

Note that the echo request in step 6 is just a simple usage example. Once the connection to the server is set up, any of the client APIs are available for use.

## NVM Operations

This section provides examples of how to use the client NVM API. Blocking APIs are used for simplicity, though the split transaction APIs can be used in a similar manner. 

Client usage of the server NVM storage first requires sending an initialization request to the server. This currently does not trigger any action on the server side but it may in the future and so it is recommended to include in client applications.

```c
int rc;
int serverRc; 
uint32_t clientId; /* unused for now */
uint32_t serverId; 

rc = wh_Client_NvmInit(&clientCtx, &serverRc, &clientId, &serverId);

/* error check both local and remote error codes */
/* serverId holds unique ID of server */
```

Once initialized, the client can create and add an object using the `NvmAddObject` functions. Note that a metadata entry must be created for every object.

```c
int serverRc;

whNvmId id = 123;
whNvmAccess access = WOLFHSM_NVM_ACCESS_ANY;
whNvmFlags flags = WOLFHSM_NVM_FLAGS_ANY;
uint8_t label[] = “My Label”;

uint8_t myData[] = “This is my data.”

whClient_NvmAddObject(&clientCtx, id, access, flags, strlen(label), &label, sizeof(myData), &myData, &serverRc);
```

Data corresponding to an existing objects can be updated in place:

```c
byte myUpdate[]	= “This is my update.”

whClient_NvmAddObject(&clientCtx, &myMeta, sizeof(myUpdate), myUpdate);
```

For objects that should not be copiend and sent over the transport, there exist DMA versions of the `NvmAddObject` functions. These pass the data to the server by reference rather than by value, allowing the server to access the data in memory directly. Note that if your platform requires custom address translation or cache invalidation before the server may access client addressses, you will need to implement a [DMA callback](TODO).

```
whNvmMetadata myMeta = {
  .id = 123,
  .access = WOLFHSM_NVM_ACCESS_ANY,
  .flags = WOLFHSM_NVM_FLAGS_ANY, 
  .label = “My Label”
};


uint8_t myData[] = “This is my data.”

wh_Client_NvmAddObjectDma(client, &myMeta, sizeof(myData), &myData), &serverRc);
```

NVM Object data can be read using the `NvmRead` functions. There also exist DMA versions of `NvmRead` functions that can be used identically to their `AddbjectDma` counterparts.

```c
const whNvmId myId = 123; /* ID of the object we want to read */
const whNvmSize offset = 0; /* byte offset into the object data */

whNvmSize outLen; /* will hold length in bytes of the requested data */
int outRc; /* will hold server return code */

byte myBuffer[BIG_SIZE];

whClient_NvmRead(&clientCtx, myId, offset, sizeof(myData), &serverRc, outLen, &myBuffer)
/* or via DMA */
whClient_NvmReadDma(&clientCtx
iint wh_Client_NvmReadDma(&clientCtx, myid, offset, sizeof(myData), &myBuffer, &serverRc);
```

Objects can be deleted/destroyed using the `NvmDestroy` functions. These funcitons take a list (array) of object IDs to be deleted. IDs in the list that are not present in NVM do not cause an error.

```c
whNvmId idList[] = {123, 456};
whNvmSize count = sizeof(myIds)/ sizeof(myIds[0]);
int serverRc;

wh_Client_NvmDestroyObjectsRequest(&clientCtx, count, &idList);
wh_Client_NvmDestroyObjectsResponse(&clientCtx, &serverRc);
```

The objects in NVM can also be enumerated using the `NvmList` functions. These functions retrieve the next matching id in the NVM list starting at `start_id`, and sets `out_count` to the total number of IDs that match `access` and `flags`:

```c
int wh_Client_NvmList(whClientContext* c,
        whNvmAccess access, whNvmFlags flags, whNvmId start_id,
        int32_t *out_rc, whNvmId *out_count, whNvmId *out_id);
```

For a full description of all the NVM API functions, please refer to the [API documentation](todo).


## Cryptography

## AUTOSAR SHE API

