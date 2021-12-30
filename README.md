# noble-secp256k1 ![Node CI](https://github.com/paulmillr/noble-secp256k1/workflows/Node%20CI/badge.svg) [![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square)](https://github.com/prettier/prettier)

[Fastest](#speed) JS implementation of [secp256k1](https://www.secg.org/sec2-v2.pdf),
an elliptic curve that could be used for asymmetric encryption,
ECDH key agreement protocol and signature schemes. Supports deterministic **ECDSA** from RFC6979 and **Schnorr** signatures from BIP0340.

[**Audited**](#security) with crowdfunding by an independent security firm. Tested against thousands of test vectors from a different library. Check out [the online demo](https://paulmillr.com/ecc) and blog post: [Learning fast elliptic-curve cryptography in JS](https://paulmillr.com/posts/noble-secp256k1-fast-ecc/)

### This library belongs to *noble* crypto

> **noble-crypto** — high-security, easily auditable set of contained cryptographic libraries and tools.

- No dependencies, one small file
- Easily auditable TypeScript/JS code
- Supported in all major browsers and stable node.js versions
- All releases are signed with PGP keys
- Check out all libraries:
  [secp256k1](https://github.com/paulmillr/noble-secp256k1),
  [ed25519](https://github.com/paulmillr/noble-ed25519),
  [bls12-381](https://github.com/paulmillr/noble-bls12-381),
  [hashes](https://github.com/paulmillr/noble-hashes)

## Usage

Use NPM in node.js / browser, or include single file from
[GitHub's releases page](https://github.com/paulmillr/noble-secp256k1/releases):

> npm install @noble/secp256k1

```js
// Common.js and ECMAScript Modules (ESM)
import * as secp from "@noble/secp256k1";
// If you're using single file, use global variable instead:
// nobleSecp256k1

(async () => {
  // You pass either a hex string, or Uint8Array
  const privateKey = "6b911fd37cdf5c81d4c0adb1ab7fa822ed253ab0ad9aa18d77257c88b29b718e";
  const messageHash = "a33321f98e4ff1c283c76998f14f57447545d339b3db534c6d886decb4209f28";
  const publicKey = secp.getPublicKey(privateKey);
  const signature = await secp.sign(messageHash, privateKey);
  const isSigned = secp.verify(signature, messageHash, publicKey);

  // Canonical signatures
  const signatureC = await secp.sign(messageHash, privateKey, { canonical: true });

  // Supports Schnorr signatures
  const rpub = secp.schnorr.getPublicKey(privateKey);
  const rsignature = await secp.schnorr.sign(messageHash, privateKey);
  const risSigned = await secp.schnorr.verify(rsignature, messageHash, rpub);
})();
```

To use the module with [Deno](https://deno.land),
you will need [import map](https://deno.land/manual/linking_to_external_code/import_maps):

- `deno run --import-map=imports.json app.ts` where `imports.json` contains the mapping from below

- `app.ts`

    ```typescript
    import * as secp from "https://deno.land/x/secp256k1/mod.ts";
    const publicKey = secp.getPublicKey("6b911fd37cdf5c81d4c0adb1ab7fa822ed253ab0ad9aa18d77257c88b29b718e");
    ```
- `imports.json`

    ```json
    {
      "imports": {
        "crypto": "https://deno.land/std@0.119.0/node/crypto.ts"
      }
    }
    ```

## API

- [`getPublicKey(privateKey)`](#getpublickeyprivatekey)
- [`getSharedSecret(privateKeyA, publicKeyB)`](#getsharedsecretprivatekeya-publickeyb)
- [`sign(hash, privateKey)`](#signhash-privatekey)
- [`verify(signature, hash, publicKey)`](#verifysignature-hash-publickey)
- [`recoverPublicKey(hash, signature, recovery)`](#recoverpublickeyhash-signature-recovery)
- [`schnorr.getPublicKey(privateKey)`](#schnorrgetpublickeyprivatekey)
- [`schnorr.sign(hash, privateKey)`](#schnorrsignhash-privatekey)
- [`schnorr.verify(signature, hash, publicKey)`](#schnorrverifysignature-hash-publickey)
- [Helpers](#helpers)

##### `getPublicKey(privateKey)`
```typescript
function getPublicKey(privateKey: Uint8Array, isCompressed?: false): Uint8Array;
function getPublicKey(privateKey: string, isCompressed?: false): string;
function getPublicKey(privateKey: bigint): Uint8Array;
```
`privateKey` will be used to generate public key.
  Public key is generated by doing scalar multiplication of a base Point(x, y) by a fixed
  integer. The result is another `Point(x, y)` which we will by default encode to hex Uint8Array.
`isCompressed` (default is `false`) determines whether the output should contain `y` coordinate of the point.

To get Point instance, use `Point.fromPrivateKey(privateKey)`.

##### `getSharedSecret(privateKeyA, publicKeyB)`
```typescript
function getSharedSecret(privateKeyA: Uint8Array, publicKeyB: Uint8Array): Uint8Array;
function getSharedSecret(privateKeyA: string, publicKeyB: string): string;
function getSharedSecret(privateKeyA: bigint, publicKeyB: Point): Uint8Array;
```

Computes ECDH (Elliptic Curve Diffie-Hellman) shared secret between a private key and a different public key.

To get Point instance, use `Point.fromHex(publicKeyB).multiply(privateKeyA)`.

To speed-up the function massively by precomputing EC multiplications,
use `getSharedSecret(privateKeyA, secp.utils.precompute(8, publicKeyB))`


##### `sign(hash, privateKey)`
```typescript
function sign(msgHash: Uint8Array, privateKey: Uint8Array, opts?: Options): Promise<Uint8Array>;
function sign(msgHash: string, privateKey: string, opts?: Options): Promise<string>;
function sign(msgHash: Uint8Array, privateKey: Uint8Array, opts?: Options): Promise<[Uint8Array | string, number]>;
```

Generates deterministic ECDSA signature as per RFC6979.

- `msgHash: Uint8Array | string` - message hash which would be signed
- `privateKey: Uint8Array | string | bigint` - private key which will sign the hash
- `options?: Options` - *optional* object related to signature value and format
- `options?.recovered: boolean = false` - whether the recovered bit should be included in the result. In this case, the result would be an array of two items.
- `options?.canonical: boolean = false` - whether a signature `s` should be no more than 1/2 prime order
- `options?.der: boolean = true` - whether the returned signature should be in DER format. If `false`, it would be in Compact format (32-byte r + 32-byte s)

The function is asynchronous because we're utilizing built-in HMAC API to not rely on dependencies.

`signSync` counterpart could also be used, you need to set `utils.hmacSha256Sync` to a function with signature `key: Uint8Array, ...messages: Uint8Array[]) => Uint8Array`. Example with `noble-hashes` package:

```ts
const { hmac } = require('noble-hashes/lib/hmac');
const { sha256 } = require('noble-hashes/lib/sha256');
secp256k1.utils.hmacSha256Sync = (key: Uint8Array, ...msgs: Uint8Array[]) => {
  const h = hmac.create(sha256, key);
  msgs.forEach(msg => h.update(msg));
  return h.digest();
};

// Can be used now
secp256k1.signSync(msgHash, privateKey)
```

##### `verify(signature, hash, publicKey)`
```typescript
function verify(signature: Uint8Array, msgHash: Uint8Array, publicKey: Uint8Array): boolean
function verify(signature: string, msgHash: string, publicKey: string): boolean
```
- `signature: Uint8Array | string | { r: bigint, s: bigint }` - object returned by the `sign` function
- `msgHash: Uint8Array | string` - message hash that needs to be verified
- `publicKey: Uint8Array | string | Point` - e.g. that was generated from `privateKey` by `getPublicKey`
- Returns `boolean`: `true` if `signature == hash`; otherwise `false`

##### `recoverPublicKey(hash, signature, recovery)`
```typescript
function recoverPublicKey(msgHash: Uint8Array, signature: Uint8Array, recovery: number): Uint8Array | undefined;
function recoverPublicKey(msgHash: string, signature: string, recovery: number): string | undefined;
```
- `msgHash: Uint8Array | string` - message hash which would be signed
- `signature: Uint8Array | string | { r: bigint, s: bigint }` - object returned by the `sign` function
- `recovery: number` - recovery bit returned by `sign` with `recovered` option
  Public key is generated by doing scalar multiplication of a base Point(x, y) by a fixed
  integer. The result is another `Point(x, y)` which we will by default encode to hex Uint8Array.
  If signature is invalid - function will return `undefined` as result.

To get Point instance, use `Point.fromSignature(hash, signature, recovery)`.

##### `schnorr.getPublicKey(privateKey)`
```typescript
function schnorrGetPublicKey(privateKey: Uint8Array): Uint8Array;
function schnorrGetPublicKey(privateKey: string): string;
```

Returns 32-byte public key. *Warning:* it is incompatible with non-schnorr pubkey.

Specifically, its *y* coordinate may be flipped. See BIP0340 for clarification.

##### `schnorr.sign(hash, privateKey)`
```typescript
function schnorrSign(msgHash: Uint8Array, privateKey: Uint8Array, auxilaryRandom?: Uint8Array): Promise<Uint8Array>;
function schnorrSign(msgHash: string, privateKey: string, auxilaryRandom?: string): Promise<string>;
```

Generates Schnorr signature as per BIP0340. Asynchronous, so use `await`.

- `msgHash: Uint8Array | string` - message hash which would be signed
- `privateKey: Uint8Array | string | bigint` - private key which will sign the hash
- `auxilaryRandom?: Uint8Array` — optional 32 random bytes. By default, the method gathers cryptogarphically secure random.
- Returns Schnorr signature in Hex format.

##### `schnorr.verify(signature, hash, publicKey)`
```typescript
function schnorrVerify(signature: Uint8Array | string, msgHash: Uint8Array | string, publicKey: Uint8Array | string): boolean
```
- `signature: Uint8Array | string | { r: bigint, s: bigint }` - object returned by the `sign` function
- `msgHash: Uint8Array | string` - message hash that needs to be verified
- `publicKey: Uint8Array | string | Point` - e.g. that was generated from `privateKey` by `getPublicKey`
- Returns `boolean`: `true` if `signature == hash`; otherwise `false`

#### Point methods

##### Helpers

###### `utils.randomPrivateKey(): Uint8Array`

Returns `Uint8Array` of 32 cryptographically secure random bytes that can be used as private key. The signature is:

```ts
(key: Uint8Array, ...msgs: Uint8Array[]): Uint8Array;
```

###### `utils.hmacSha256Sync`

The function is not defined by default, but could be used to implement `signSync` method (see above).

###### `utils.precompute(W = 8, point = BASE_POINT): Point`

Returns cached point which you can use to pass to `getSharedSecret` or to `#multiply` by it.

This is done by default, no need to run it unless you want to
disable precomputation or change window size.

We're doing scalar multiplication (used in getPublicKey etc) with
precomputed BASE_POINT values.

This slows down first getPublicKey() by milliseconds (see Speed section),
but allows to speed-up subsequent getPublicKey() calls up to 20x.

You may want to precompute values for your own point.

```typescript
secp256k1.CURVE.P // Field, 2 ** 256 - 2 ** 32 - 977
secp256k1.CURVE.n // Order, 2 ** 256 - 432420386565659656852420866394968145599
secp256k1.Point.BASE // new secp256k1.Point(Gx, Gy) where
// Gx = 55066263022277343669578718895168534326250603453777594175500187360389116729240n
// Gy = 32670510020758816978083085130507043184471273380659243275938904335757337482424n;

// Elliptic curve point in Affine (x, y) coordinates.
secp256k1.Point {
  constructor(x: bigint, y: bigint);
  // Supports compressed and non-compressed hex
  static fromHex(hex: Uint8Array | string);
  static fromPrivateKey(privateKey: Uint8Array | string | number | bigint);
  static fromSignature(
    msgHash: Hex,
    signature: Signature,
    recovery: number | bigint
  ): Point | undefined {
  toRawBytes(isCompressed = false): Uint8Array;
  toHex(isCompressed = false): string;
  equals(other: Point): boolean;
  negate(): Point;
  add(other: Point): Point;
  subtract(other: Point): Point;
  // Constant-time scalar multiplication.
  multiply(scalar: bigint | Uint8Array): Point;
}
secp256k1.Signature {
  constructor(r: bigint, s: bigint);
  // DER encoded ECDSA signature
  static fromDER(hex: Uint8Array | string);
  // R, S 32-byte each
  static fromCompact(hex: Uint8Array | string);
  toDERRawBytes(): Uint8Array;
  toDERHex(): string;
  toCompactRawBytes(): Uint8Array;
  toCompactHex(): string;
}
```

## Security

Noble is production-ready.

1. The library has been audited by an independent security firm cure53: [PDF](https://cure53.de/pentest-report_noble-lib.pdf). The audit has been [crowdfunded](https://gitcoin.co/grants/2451/audit-of-noble-secp256k1-cryptographic-library) by community with help of [Umbra.cash](https://umbra.cash).
2. The library has also been fuzzed by [Guido Vranken's cryptofuzz](https://github.com/guidovranken/cryptofuzz). You can run the fuzzer by yourself to check it.

We're using built-in JS `BigInt`, which is "unsuitable for use in cryptography" as [per official spec](https://github.com/tc39/proposal-bigint#cryptography). This means that the lib is potentially vulnerable to [timing attacks](https://en.wikipedia.org/wiki/Timing_attack). But, *JIT-compiler* and *Garbage Collector* make "constant time" extremely hard to achieve in a scripting language. Which means *any other JS library doesn't use constant-time bigints*. Including bn.js or anything else. Even statically typed Rust, a language without GC, [makes it harder to achieve constant-time](https://www.chosenplaintext.ca/open-source/rust-timing-shield/security) for some cases. If your goal is absolute security, don't use any JS lib — including bindings to native ones. Use low-level libraries & languages. Nonetheless we've hardened implementation of koblitz curve multiplication to be algorithmically constant time.

We however consider infrastructure attacks like rogue NPM modules very important; that's why it's crucial to minimize the amount of 3rd-party dependencies & native bindings. If your app uses 500 dependencies, any dep could get hacked and you'll be downloading rootkits with every `npm install`. Our goal is to minimize this attack vector.

## Speed

Benchmarks measured with Apple M1.

    getPublicKey(utils.randomPrivateKey()) x 6,121 ops/sec @ 163μs/op
    sign x 4,679 ops/sec @ 213μs/op
    verify x 923 ops/sec @ 1ms/op
    recoverPublicKey x 491 ops/sec @ 2ms/op
    getSharedSecret aka ecdh x 534 ops/sec @ 1ms/op
    getSharedSecret (precomputed) x 7,105 ops/sec @ 140μs/op
    Point.fromHex (decompression) x 12,171 ops/sec @ 82μs/op
    schnorr.sign x 409 ops/sec @ 2ms/op
    schnorr.verify x 504 ops/sec @ 1ms/op

Compare to other libraries (`openssl` uses native bindings, not JS):

    elliptic#getPublicKey x 1,940 ops/sec
    sjcl#getPublicKey x 211 ops/sec

    elliptic#sign x 1,808 ops/sec
    sjcl#sign x 199 ops/sec
    openssl#sign x 4,243 ops/sec
    ecdsa#sign x 116 ops/sec
    bip-schnorr#sign x 60 ops/sec

    elliptic#verify x 812 ops/sec
    sjcl#verify x 166 ops/sec
    openssl#verify x 4,452 ops/sec
    ecdsa#verify x 80 ops/sec
    bip-schnorr#verify x 56 ops/sec

    elliptic#ecdh x 971 ops/sec


## Contributing

Check out a blog post about this library: [Learning fast elliptic-curve cryptography in JS](https://paulmillr.com/posts/noble-secp256k1-fast-ecc/).

1. Clone the repository.
2. `npm install` to install build dependencies like TypeScript
3. `npm run compile` to compile TypeScript code
4. `npm run test` to run jest on `test/index.ts`

Special thanks to [Roman Koblov](https://github.com/romankoblov), who have helped to improve scalar multiplication speed.

## License

MIT (c) Paul Miller [(https://paulmillr.com)](https://paulmillr.com), see LICENSE file.
