---
caip: 275
title: Domain Wallet Authentication
author: Chris Cassano (@glitch003), David Sneider (@davidlsneider), Federico Amura (@FedericoAmura), Gregory Markou (@GregTheGreek)
discussions-to: https://github.com/ChainAgnostic/CAIPs/pull/275
status: Draft
type: Standard
created: 2024-04-29
updated: 2024-05-13
---

# Abstract

The Domain Wallet Authentication describes a method for linking a crypto domain with authentication methods or providers by adding an authenticator: JSON/URL field to the metadata of a crypto domain NFT.
The standard also describes a method for application developers and web3 login modal providers to enable users to login with their domain name.

The goal of this specification is to define a chain-agnostic identity standard for domain-based authentication.

# Terminology

- Crypto Domain: A domain name that is associated with a blockchain address, typically stored in a blockchain NFT.
- Crypto Domain NFT: A non-fungible token that represents ownership of a crypto domain.
- Wallet Provider: A service that provides wallet functionality, such as signing transactions or messages.
- Text record: A record that stores text-based information, such as an authenticator URL or JSON object.
- Domain lookup/query: The process of retrieving information associated with a domain name, such as an address or text record.
- Authenticator: A URL or JSON object that provides authentication details or additional information related to a domain name.
- Authenticator Flow provider: Any service that will store Authentication Flows linked to Crypto Domains
- Authentication Flow: Configuration for the requesting application to find and connect to the wallet that it wants to authenticate with

# Motivation

Current blockchain authentication methods primarily rely on connecting wallets via providers like Metamask.
However, this requires users to remember both their wallet provider and their wallet address.
As more wallets, signers, and chains come online, this problem will only get worse.

Crypto domains provide a human-readable, user-friendly way to represent wallet addresses.
By enabling authentication directly with crypto domains, this standard aims to improve usability and adoption of web3 logins.

Additionally, standardizing the way domain NFT metadata specifies its supported authentication mechanisms allows any compatible domain NFT to abstract out authentication methods and key management.
This abstraction allows both login modals and dApps to easily integrate domain-based logins.

# Specification

## Storage Format

Any system capable of resolving text records can be used for the name in this system.
However, we have chosen to focus this specification on Crypto domain NFTs.
Crypto domain NFTs that are compatible with this authentication standard MUST include an `authenticator` text record entry with the following properties:
`authenticator` (string, required): An URL that dereferences to a JSON object containing configuration information.
In particular, information about how to authenticate the domain's subject. e.g.

`https://www.authprovider.com/auth/{}`

The application will craft the final URL to get the configuration, where "{}" will be substituted for the user's whole crypto domain name, so for "chrisc.eth" the final URL would be `http://www.authprovider.com/auth/chrisc.eth`

The actual syntax used to store this `authenticator` text record will vary depending on that of the particular Crypto Domain system used.
For example, on ENS, a [ENSIP-5][] record should be created, with a `key` of `authenticator`.
Any relying party can then query the domain for the `authenticator` text record via the [ENSIP-5][] getter interface, i.e. `text(bytes32 node, string key)` where `node` is the ENS domain being queried and `key` is the string "authenticator".

### Example of a request to retrieve the "authenticator" ENS text record:

```typescript
import { normalize } from "viem/ens";
import { publicClient } from "./client";

const userDomain = "chrisc.eth";

const authenticatorRecord = await publicClient.getEnsText({
  name: normalize(userDomain),
  key: "authenticator",
});
// ensText will be 'https://www.authprovider.com/auth/{}'

const authenticationFlowsURL = authenticatorRecord.replace("{}", userDomain);
// authenticationFlowsURL will be 'https://www.authprovider.com/auth/chrisc.eth'
```

## Authentication flows retrieval

Having the authentication flows URL, the application can perform an HTTP GET request to it and obtain their configuration.
Application will filter the authentication flows that are not supported and then try to execute in order, passing on the ones that cannot be fulfilled until it finds one that is successful or needs action from the user.

An authentication flow may be unsupported due to the platform it is running mismatching the one it requires, e.g. "browser", "mobile", etc. Or because it requires an specific wallet but it is not installed, e.g. requires "io.metamask" but there is no browser extension wallet.
When an authentication flow requires user action, such as scanning a QR code, or authorizing the connection from the wallet, the application must wait for the user to complete or cancel the action.

### Authentication flows definition

The Authentication flows definition JSON MUST conform to the following [Draft 7 JSON Schema][]:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "address": {
      "type": "string",
      "description": "The address requested by the user. This can be from any chain."
    },
    "chain": {
      "type": "string",
      "description": "The blockchain id as specified in CAIP-2. This is optional and can be used to filter the authentication flows by chain. If not provided, it should support all chains that support the address."
    },
    "authFlows": {
      "type": "array",
      "description": "List of authentication flows supported for different platforms and connections. At least one value must be provided.",
      "items": {
        "type": "object",
        "properties": {
          "platform": {
            "type": "string",
            "description": "The platform type, e.g., 'browser' or 'mobile'. If not specified, it should support all platforms"
          },
          "connection": {
            "type": "string",
            "description": "The method of connection, e.g., 'extension', 'wc' (wallet connect), or 'mwp' (mobile wallet protocol)."
          },
          "URI": {
            "type": "string",
            "description": "The Uniform Resource Identifier used to initiate the authentication process. This can be a URL with an optional domain placeholder, a universal link, a wallet rdns following EIP-6963 or the string 'injected'. E.g., 'io.metamask', 'https://domainwallet.io/wallet', etc. In case this value is not provided it will default according to the application criteria, e.g. looking for an 'injected' wallet in browser"
          }
        },
        "required": ["connection"]
      }
    }
  },
  "required": ["address", "authFlows"]
}
```

Possible values for each field are:

- `platform`:
  - `browser`
  - `mobile`
  - undefined
- `connection`:
  - `extension`, when
  - `wc`, to trigger a connection with the wallet using [Wallet Connect][]
  - `mwp`, to discover and communicate with the wallet usign [Mobile Wallet Protocol][]

Those values are not exhaustive. Meaning they can be extended as new options arise or become popular. For example, a new platform such as `wearable` or a new communication protocol could be defined

To further clarify how authentication flows should be used, let's consider the following example:

```json
{
  "address": "0xd8da6bf26964af9d7eed9e03e53415d37aa96045",
  "authFlows": [
    {
      "platform": "browser",
      "connection": "extension",
      "URI": "com.wallet.rdns"
    },
    {
      "platform": "mobile",
      "connection": "mwp",
      "URI": "https://universal-link.wallet.com/"
    },
    {
      "connection": "wc",
      "URI": "http://www.wallet.com/auth"
    }
  ]
}
```

To resolve that configuration:

1. Before all, application can validate that the returned `address` is in fact the one that the Crypto Domain resolved previously
2. Application will iterate the authentication flows, automatically discarding the ones that are not supported until one is completed or requires user action.
3. First authentication flow indicates that is has to run on `browser`, where there has to be an `extension` installed with the resource identifier `com.wallet.rdns` (not any extension. For such case, "injected" should be used and it application). When this authentication flow is possible, application will try to find the wallet and request a connection using the standard mechanism to communicate with browser extensions wallets.
4. Second authentication flow indicates that it has to run on `mobile`, and using [Mobile Wallet Protocol][] (indicated with the key `mwp`) application has to find another app that responds to the universal link `https://universal-link.wallet.com/`
5. Last authentication flow does not impose a platform to run on because it does not specify any, so it will be valid everywhere. It tries to stablish a [Wallet Connect][] connection to the URI `http://www.wallet.com/auth`. When this authentication flow is triggered, the URI will be opened in a new browser window passing `address`, `domain` and `wcUri` as query params with the name resolved address, the domain name and [Wallet Connect][] URI respectively. As this flow requires user action (connecting the wallet at the URI) this step will never be skipped as long as the application supports [Wallet Connect][].

### Other examples

To further clarify different authentication flow configurations we can consider the following cases

#### Use the wallet extension installed at the browser, without requiring a specific one

```json
{
  "platform": "browser",
  "connection": "extension",
  "URI": "injected"
}
```

As this authentication flow calls for the `injected` extension, without specifying which one, it should use EIP-6963 to check if a matching extension provider is available, and use that if it is available. If one is not found, we resolve to the default one.
For ethereum addresses, this would be `window.ethereum`.
However, this configuration is not recommended as having multiple wallets generates a race condition and can provide false positives where there is a wallet extension but it does not have the account linked to the domain. We recommend EIP-6963 if possible.

#### Wallet connect to any wallet

```json
{
  "connection": "wc"
}
```

Using this configuration, the application should provide the QR code and [Wallet Connect][] URI for the user to connect with his wallet from any place.
Such configuration can be useful when the user has their wallet in their mobile device and also on another platform such as a web wallet or desktop application that can receive the QR code or [Wallet Connect][] URI in any way and they want to decide which one to use at connection time.

This configuration will never be skipped as it does not impose any platform and requires the user action of connecting from the wallet making this a great last case in the `authFlows` array and behaving as the default when no other authentication flow is useful.

## Self hosting authentication flows

In case the user does not trust any authentication flow provider to store their info, they can easily host it themselves using a public serverless function and saving its URL in the ENS `authenticator` record.

## Direct authentication flows resolution

There are a few cases where the `authenticator` record can be set to the authentication flows directly

- The user wants a truly decentralized solution and does not care about the gas cost of storing lots of data in blockchain; then they can write a stringified version of their authentication flows definition in the `authenticator` text record
- Application will resolve only its issued names, therefore the authentication flows or their location are likely already known. In such cases the application will likely use its own backend to resolve `authenticator` instead of ENS or another Crypto Domain system NFT system. Responding with the stringified version of the saves the user of the extra request as it does not provide any useful information.

# Login With Name Flow

![sequenceDiagram](../assets/CAIP-X/sequenceDiagram.png)

Web3 applications and login modal providers can implement the Login With Name flow following the sequence pictured in the diagram above:

1. Users starts the Login With Name process
2. Application requests name and user enters a name linked with the address of the wallet they control and want to authenticate with.
3. Application uses its configured name resolvers to resolves the name and its linked authentication flows under the `authenticator` text record.
   If the authenticator text record is a URL, client application sends an HTTP GET request to it in order to obtain users authentication flows.
   This is not needed if the authenticator text record already has a stringified JSON in it.
4. (Optional) Validate the response from name resolver is for the same address resolved by the Crypto Domain NFT.
5. Application filters invalid authentication flows and initiates the one that can be completed which could be by opening a new window in the user's browser to the authenticator URL, triggering the browser wallet, etc.
   If it is not supported due to platform or some other requirement, it can continue with the next one
6. If no authentication flow can be processed by the application, then display this situation accordingly to the user.
7. Wallet, when needed, will trigger the user for authentication and connection approval before stablishing the connection with the wallet.
8. After wallet approves connection, application can match the domain name, the user's wallet address, and an authenticated session to the ones requested previously to validate user is not connecting with another wallet.
9. When everything succeeds, connection should be stablished between application and the wallet found and triggered under the name provided by the user.

## Connection properties

How the dApp actually requests signatures and talks to the signer will vary depending on the authenticator and the chain being used.
One option for session management is WalletConnect, where the "authenticated session" returned is actually a WalletConnect session.
Applications may also choose to directly integrate signer SDKs, providing a more streamlined signing flow.

For example, here is how an application would integrate with the Login With Name login process:

1. Install wagmi and [Login With Name WAGMI connector][] packages
2. Import the loginWithName connector from the package
3. Configure the wagmi config with the loginWithName connector.
   This package provides ENS as a name resolver which can be configured to use mainnet or sepolia
4. Wrap the app with WagmiProvider, pass it the configuration with the loginWithName connector
5. Upon connection request, the connector will request application for user name. Application can show user an input for it if not obtained in other way.
6. Connector will now resolve user name into an address and authentication flows and trying to reach the wallet specified and its accounts.
7. After signing in with the wallet. Application can use standard wagmi hooks like useAccount, useConnect etc.

## Name Resolver systems

For the purposes of this document, we've detailed a flow based on ENS domains.
But this standard is extensible to any name resolution system.
For example, to use Solana domains, you could just replace the ENS name resolver component with a Solana name resolver component.
The name resolver can also mix several sources, so resolving using several chains is only a matter of combining the specific name resolver for each chain and then combining them with the necessary logic

Reference implementation included in [Login With Name WAGMI connector][] include two name resolvers to showcase different implementations.

- ENS, compatible with any chain that uses it and likely the one that will be the most common
- Domain Wallets, an online web wallet system compatible with many chains and that already integrates a domain naming system to identify each wallet
- An application level mixer name resolver, that combines the two previous ones

It has to be noted too that the name resolver can be any service that resolves names and their associated `authenticator` text record to an address and authentication flows respectively.
A name resolver can take any type of name and as long as it can resolve it, it does not have to be tied to any chain. Also they can provide their own logic to resolve conflicts where a name exists on different chains or services
For example, name resolvers could take names from:

- Crypto NFT domain names such as ENS
- Social networks (facebook, X, Lens, etc.)
- Emails

And define the order or resolution themselves, totally transparent to the application that uses them.
If the application wants to impose their own logic for resolution over theirs, then it can combine several name resolvers as they all have to adhere to the same interface as shown in the demo.

### Function to Resolve a Domain Name to an Address

> Function: resolveName
> Description: Resolves a given domain name to a blockchain address.
> Input: name (String) - The domain name to be resolved.
> Output: Address (String | null) - The blockchain address associated with the domain name, or null if no address is found.

This function takes a string input representing the domain name and returns either the associated blockchain address as a string or null if the address cannot be found.
This function must handle various types of blockchain addresses and be adaptable to different blockchain technologies.

### Function to Resolve a Domain Name to an authenticator URL or JSON

> Function: resolveAuthenticator
> Description: Resolves a given domain name to a URL or a JSON object that can be used for authentication or further information retrieval.
> Input: name (String) - The domain name to be resolved.
> Output: Authenticator (String | JSON | null) - A URL or JSON object providing authentication details or additional information, or null if no data can be found.

This function accepts a domain name as a string and returns a URL or a JSON object.
The output is intended to provide authentication details or additional information related to the domain name.
If no relevant data can be found, the function returns null.
This function should be flexible enough to support different formats and data structures, adapting to the needs of various blockchain platforms and applications.

#### ENS example implementation using Typescript and Viem:

```typescript
import {
  type Address,
  type Chain,
  createPublicClient,
  http,
  PublicClient,
} from "viem";
import { mainnet } from "viem/chains";
import { normalize } from "viem/ens";

export interface NameResolver {
  resolveName(name: string): Promise<Address | null>;
  resolveAuthenticator(name: string): Promise<string | null>;
}

export interface ENSOptions {
  chain?: Chain;
  jsonRpcUrl?: string;
}

export class ENS implements NameResolver {
  private readonly client: PublicClient;

  constructor(options: ENSOptions) {
    this.client = createPublicClient({
      chain: options.chain ?? mainnet,
      transport: http(options.jsonRpcUrl),
    });
  }

  async resolveName(domainName: string): Promise<Address | null> {
    return this.client.getEnsAddress({
      name: normalize(domainName),
    });
  }

  async resolveKey(domainName: string, key: string): Promise<string | null> {
    return this.client.getEnsText({
      name: normalize(domainName),
      key,
    });
  }

  async resolveAuthenticator(domainName: string): Promise<string | null> {
    return this.resolveKey(domainName, "authenticator");
  }
}
```

# Rationale

Specifying the authenticator URL as a domain name NFT text record allows applications to easily discover and integrate with compatible login methods in a standard way
Having a chain-agnostic standard enables interoperability between different crypto domain providers and authentication methods
Providing clear wallet implementer steps and code samples makes it easy for developers to adopt this standard

# Backwards Compatibility

This standard is fully backwards compatible as it proposes an additional metadata field for crypto domain NFTs.
Existing NFTs and applications will continue to function normally.

## Reference Implementation

A demo wagmi connector is published at [Login With Name WAGMI connector][] public repository.
Also and a working demo with further details on how this CAIP works is hosted at [the following link](https://login-with-name-wagmi-sdk.vercel.app/)

## References

- [CAIP-2][] - Chain Agnostic Identity Protocol
- [EIP-1193][] - Ethereum Provider JavaScript API
- [EIP-4361][] - Sign-In with Ethereum
- [EIP-6963][] - Multi Injected Provider Discovery
- [ENSIP-5][] - ENS Text Records
- [Draft 7 JSON Schema][] - Draft 7 JSON Schema
- [Login With Name WAGMI connector][] - Login With Name WAGMI connector
- [Mobile Wallet Protocol][] - Mobile Wallet specification
- [Wallet Connect][] - Wallet Connect SDK and docs

[CAIP-2]: https://chainagnostic.org/CAIPs/caip-2
[EIP-1193]: https://eips.ethereum.org/EIPS/eip-1193
[EIP-4361]: https://eips.ethereum.org/EIPS/eip-4361
[EIP-6963]: https://eips.ethereum.org/EIPS/eip-6963
[ENSIP-5]: https://docs.ens.domains/ensip/5
[Login With Name WAGMI connector]: https://github.com/FedericoAmura/login-with-name-wagmi-sdk
[Draft 7 JSON Schema]: http://json-schema.org/draft-07/schema#
[Mobile Wallet Protocol]: https://mobilewalletprotocol.github.io/wallet-mobile-sdk/
[Wallet Connect]: https://walletconnect.com/

# Copyright

Copyright and related rights waived via [CC0](https://github.com/ChainAgnostic/CAIPs/blob/main/LICENSE).
