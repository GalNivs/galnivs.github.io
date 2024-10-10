---
layout: post
title: "EVM Vulnerabilities: From Precompiles To Node Takeover and Mass Destruction"
toc: true
---

![EVM Blow](/assets/img/evm_blow.jpg)

## Introduction

Hello, my name is Gal Niv, and I am a 24-year-old security researcher. I enjoy exploring new and fun technologies in my free time, and I’ve done so with blockchains, among other things that I plan to post about here soon.

### Motivation

I wanted to challenge myself, so I looked for interesting "attack surfaces" in blockchain infrastructure.

While reading about blockchains, it struck me quite quickly that the area of Ethereum's middleware (or any blockchain, for that matter) is often overlooked, and not many in the web3 community even understand its internals. Furthermore, when examining bug bounty programs, the biggest middleware programs (like geth's bug bounty program, for example) offer much lower rewards than those for protocols (like Immunefi's biggest ones, for example).

So, what exactly is "middleware" in this context? I would say it’s all the code connecting web3 with "web2" and native computers. Under that definition, in Ethereum, middleware code includes the node software, as it manages blockchain data, processes transactions, and provides APIs for decentralized applications to interact with the network. Some might also say that crypto wallets can be considered middleware since they facilitate the interaction between users and the blockchain by generating private keys, managing private keys, signing transactions, and communicating with blockchain nodes ("web2" communication).

The primary attack surface in an Ethereum node is focused on the RPC interface and the Ethereum Virtual Machine (EVM), as these components handle external interactions and execute VM opcodes, respectively, making them sophisticated and interesting targets. I decided to focus on EVM implementations and, later on, specifically on precompiles, as they seemed to me like a very awkward addition to the existing EVM design.

For those who need a bit of background...

### Background

#### What is EVM?

Ethereum Virtual Machine (EVM) is the runtime environment for smart contracts in Ethereum. In other words, it is yet another virtual machine, specifically designed for on-chain code execution. The EVM allows developers to write code compiled into "smart contracts," which are self-executing on demand (via Ethereum transactions). This VM is built out of [1-byte opcodes](https://evm.codes) and is Turing complete, allowing for sophisticated programs.

#### What are Precompiles?

Another fundamental concept of the EVM is what's called [precompiles](https://evm.codes/precompiled). These are a special kind of contract bundled with the EVM at fixed addresses. Essentially, these contracts provide services that would be expensive or not even feasible to implement in EVM bytecode. In other words, mathematical primitives (elliptic curve cryptography, hash functions, etc.) are provided and natively implemented within nodes in this way for smart contracts to use.

#### What is a Side-Chain?

A side-chain is a separate blockchain that is connected to and operates in parallel with a main blockchain, such as Ethereum. The purpose of a side-chain is to enable faster, cheaper, and more scalable transactions by offloading some of the transaction load from the main blockchain.
In the following part, we will examine a project that utilizes the side-chain concept.

## Hunting EVM Vulnerabilities

The ecosystem of blockchains evolved entirely in the modern age, and as such, its core is mostly implemented very neat.

Most infrastructure code is implemented in Go or Rust, with native code mainly consisting of standard libraries and small codebases like [libsecp256k1](https://github.com/bitcoin-core/secp256k1).

At this point the world of Ethereum nodes pretty much consolidated into [go-ethereum (AKA geth)](https://github.com/ethereum/go-ethereum) node implementation.

However, new projects and protocols in web3 often introduce mistakes and vulnerabilities despite Ethereum's robust and secure core code due to complexity of novel features.

One interesting example in the attack surface we're going to discuss is [Zellic's Astar research](https://www.zellic.io/blog/finding-a-critical-vulnerability-in-astar/) , which uncovered a vulnerability in such a mechanism—an added precompile that was prone to an integer truncation vulnerability.

I conducted my research in 2022 but have never gotten around to writing this blog until now, so we’re going back to a time before the case mentioned above.

### SKALE and EVM

#### What is SKALE Network

The SKALE Network is an Ethereum-based platform designed to enhance the scalability of blockchain networks. It provides low-cost sidechains in the payment model of rental.

So essentially SKALE has "managers" on-chain, which are being paid a fee for sidechain rental and then transactions on the sidechain are free of charge. Pretty neat idea!

[SKALE's whitepaper](https://skale.space/whitepaper) shows all the hefty details, SKALE holds a validators pool which are being allocated for different sidechains at a time.

The project is constantly developing and has awesome community as well as a whole lot of unique features, in this drill down we are going to look at its filesystem feature.

![skale ethereum manniet](/assets/img/skale.webp)*Diagram taken from [SKALE Primer](https://https://skale.space/primer)*

#### FileStorage and SKALE

Quoting SKALE's whitepaper about its storage features:

``` markdown
"To expand potential use-cases, SKALE has modified the existing EVM to allow
 for much larger file storing capabilities. The changes enabling this included
 the increase of block sizes (allowing for more data to be included in each
 block) as well as direct access to each node’s file system from a fileStorage
 precompiled smart contract."
```

First two thoughts that went through my mind when reading this:

1. Sounds super versatile and cool feature
2. Direct access to the node's file system from a precompile?! what the...

So just a quick brief on how that feature works, the [FileStorage documentation](https://docs.skale.network/tools/filestorage/) got it pretty good:

![filestorage architecture](https://docs.skale.network/tools/_images/filestorage-architecture.svg)

`Filestorage.js` is a js wrapper to communicate with SKALE sidechains,
`Filestorage.sol` is wrapping the precompiles for comfortable apis,
`Precompiled.cpp` implements the precompiles natively for fs access.

Great, seems like we have a good candidate feature to look at, messing with filesystem is always tough and the complexity of web3 must make the challenge even bigger.

#####  Drill Downs and Vulnerabilities

The information above reveals two interesting layers for us to look at:

1. The Solidity contract (`Filestorage.sol`)
2. The node implementation (`Precompiled.cpp`)

###### Contract Drill Down

This contract has the actual `external` functions that users call to use the FileStorage feature.

While examining the code an interesting flow was found when making calls to `createDirectory` and `deleteDirectory`:

``` solidity
function createDirectory(string calldata directoryPath) external {

    // ............ truncated code

    string[] memory dirs = Utils.parseDirectoryPath(directoryPath); // <-------- [1]
    Directory storage currentDirectory = rootDirectories[owner];
    for (uint i = 1; i < dirs.length; ++i) {
        require(currentDirectory.contentIndexes[dirs[i - 1]] > EMPTY_INDEX, "Invalid path");
        currentDirectory = currentDirectory.directories[dirs[i - 1]];
    }

    // ............ truncated code

    string memory newDir = (dirs.length > 1) ? dirs[dirs.length - 1] : directoryPath;
    require(Utils.checkContentName(newDir), "Invalid directory name"); // <-------- [2]
    bool success = PrecompiledCaller.createDirectory(owner, directoryPath);
    require(success, "Directory not created");
    ContentInfo memory directoryInfo = ContentInfo({
        name: newDir,
        isFile: false,
        size: 0,
        status: FileStatus.NONEXISTENT,
        isChunkUploaded: new bool[](0)
    });
    currentDirectory.contents.push(directoryInfo);
    currentDirectory.contentIndexes[newDir] = currentDirectory.contents.length;
    occupiedStorageSpace[owner] += directoryFsSize;
}

function deleteDirectory(string calldata directoryPath) external {

    // ............ truncated code

    string[] memory dirs = Utils.parseDirectoryPath(directoryPath);
    Directory storage currentDirectory = rootDirectories[owner];
    for (uint i = 1; i < dirs.length; ++i) {
        require(currentDirectory.contentIndexes[dirs[i - 1]] > EMPTY_INDEX, "Invalid path");
        currentDirectory = currentDirectory.directories[dirs[i - 1]];
    }

    // ............ truncated code

    bool success = PrecompiledCaller.deleteDirectory(owner, directoryPath); // <-------- [3]
    require(success, "Directory is not deleted");
    ContentInfo memory lastContent = currentDirectory.contents[currentDirectory.contents.length - 1];
    currentDirectory.contents[currentDirectory.contentIndexes[targetDirectory] - 1] = lastContent;
    currentDirectory.contentIndexes[lastContent.name] = currentDirectory.contentIndexes[targetDirectory];
    currentDirectory.contentIndexes[targetDirectory] = EMPTY_INDEX;
    currentDirectory.contents.pop();
    // slither-disable-next-line mapping-deletion
    delete currentDirectory.directories[targetDirectory];
    occupiedStorageSpace[owner] -= Utils.calculateDirectorySize();
}
```

**Vulnerability 1 - FileStorage Destruction**

In `createDirectory`, the full path passed from user input is firstly checked that all parent directories already in existence in [1].
This happens with `parseDirectoryPath` which splits the given directory by `'/'` creating the list called `dirs`.
Later on when the code tries to actually create the directory, the full path is used again in `newDir`.
`newDir` is checked for dir traversals in [2] with the help of `checkContentName`:

``` solidity
function checkContentName(string memory contentName) internal pure returns (bool) {
    if (keccak256(abi.encodePacked(contentName)) == keccak256(abi.encodePacked("..")) ||
    keccak256(abi.encodePacked(contentName)) == keccak256(abi.encodePacked(".")) ||
    bytes(contentName).length == 0) {
        return false;
    }
    uint nameLength = bytes(contentName).length;
    if (nameLength > MAX_FILENAME_LENGTH) {
        return false;
    }
    return true;
}
```

This function checks that the directory is explicitly not `"."`{: .filepath} nor `".."`{: .filepath}, this seems to be fine because in the case of passing full path with any directory traversal, the function would already fail in [1] (after splitting the path, there will be no entry for `..` and it can not be created).

As for `deleteDirectory`, the full path passed from user input gets deleted in [3], this will be used later.

The issue with the functions described above is with the edge case of the path `"../"`{: .filepath} passed to `createDirectory` since it will eliminate the checks in `checkContentName` (`"../" != ".."`).

Since the paths are split by `'/'`{: .filepath} in every file or directory creation the new `"../"`{: .filepath} entry can not be abused to a full traversal, but an attacker could call `deleteDirectory` with the `"../"`{: .filepath} path leading to the deletion of all FileStorage folders!

###### Precompiled Drill Down

As the file name implies, the relevant precompiles are written in c++ and integrated within SKALE's node implementation called [skaled](https://github.com/skalenetwork/skaled) (a fork of cpp-ethereum project).

The [file mentioned above](https://github.com/skalenetwork/skaled/blob/develop/libethereum/Precompiled.cpp) has quite a few precompiles for the FileStorage feature - `createFile`, `uploadChunk`, `readChunk`, `getFileSize`, `deleteFile`, `createDirectory`, `deleteDirectory`, `calculateFileHash`.

Taking a look at all of them reveals two main problems, we'll examine `readChunk` for example:

``` cpp
ETH_REGISTER_PRECOMPILED( readChunk )( bytesConstRef _in ) {
    MICROPROFILE_SCOPEI( "VM", "readChunk", MP_ORANGERED );
    try {
        
    // ............ truncated code
        
        size_t filenameLength;
        std::string filename;
        convertBytesToString( _in, 32, filename, filenameLength ); // <-------- [3]
        size_t const filenameBlocksCount = ( filenameLength + UINT256_SIZE - 1 ) / UINT256_SIZE;

        bigint const bytePosition( parseBigEndianRightPadded(
            _in, 64 + filenameBlocksCount * UINT256_SIZE, UINT256_SIZE ) );
        size_t const position = bytePosition.convert_to< size_t >(); // <-------- [2]

        bigint const byteChunkLength( parseBigEndianRightPadded(
            _in, 96 + filenameBlocksCount * UINT256_SIZE, UINT256_SIZE ) );
        size_t const chunkLength = byteChunkLength.convert_to< size_t >(); // <-------- [2]

        const fs::path filePath = getFileStorageDir( Address( address ) ) / filename; // <-------- [1]
        if ( position > stat_compute_file_size( filePath.c_str() ) ) {
            throw std::runtime_error(
                "readChunk() failed because chunk gets out of the file bounds" );
        }

        std::ifstream infile( filePath.string(), std::ios_base::binary );
        infile.seekg( static_cast< long >( position ) );
        bytes buffer( chunkLength );
        infile.read(
            reinterpret_cast< char* >( &buffer[0] ), static_cast< long >( buffer.size() ) );
        return { true, buffer };
  
    // ............ truncated code

    return { false, response };
}
```

**Vulnerability 2 - Directory Traversal**

First and foremost [1] shows a native code that directly uses `filename` which comes from caller input, one might send a `filename` with directory traversal (e.g. `"../../../../../../etc/password"`{: .filepath}) and read/write out of its contract's folders.
Using SKALE's IMA SDK I was able to create a test node locally and successfully exploit the issue.

It was later brought to my attention by SKALE team that in production chains, the default config includes a key-value that restricts access to the fs write functionality for the chain owner only (`"restrictAccess": ["fs"]`).
Unfortunately for `readChunk` that is not the case since filestorage assets are intended to be publicly accessible allowing arbitrary file read from the traversal mentioned above. Chain owners can still even takeover the host nodes that run their side-chain.

**Vulnerability 3 - Memory Corruptions**

As for [2] and [3], a value is taken from user input into a `size_t` argument which is later used without validation.
The `seek` and `read` values for example used from the input of [2] (potentially causing uninit read and allocation failure).
[3] is a variant of the same issue embedded within `convertBytesToString` implementation:

``` c++ 
static void convertBytesToString(
    bytesConstRef _in, size_t _startPosition, std::string& _out, size_t& _stringLength ) {
    bigint const sstringLength( parseBigEndianRightPadded( _in, _startPosition, UINT256_SIZE ) );
    _stringLength = sstringLength.convert_to< size_t >(); // <-------- [1]
    vector_ref< const unsigned char > byteFilename =
        _in.cropped( _startPosition + 32, _stringLength );
    _out = std::string( ( char* ) byteFilename.data(), _stringLength );
}
```

We can see the same problematic pattern in [1], which is passed to the `cropped` function of a `bytesConstRef` object. This results with a null field in the `_out` param (`byteFilename.data() = NULL`) and causes a null-dereference.

## Bounty and Thanks

I reported the vulnerability through SKALE's HackerOne and after a short triage (+ a very good talk with the relevant team) I received 10,250$ bounty for the three vulnerabilities reported.

I would personally like to thank SKALE team for taking the issues very seriously, deploying fast patches and paying a very respectable bug bounty albeit being out of scope for their official bug bounty program on HackerOne, you guys are awesome!

