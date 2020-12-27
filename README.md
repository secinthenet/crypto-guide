# Cryptocoins management guide

> ⚠️ **NOTE**: WIP.

## Table of Contents

-   [Table of Contents](#table-of-contents)
-   [Goals](#goals)
    -   [Non-Goals](#non-goals)
-   [Disclaimer](#disclaimer)
-   [Principles](#principles)
-   [Threat model](#threat-model)
-   [Terminology](#terminology)
    -   [BIP32 seed](#bip32-seed)
    -   [BIP32 root key](#bip32-root-key)
    -   [BIP39 entropy](#bip39-entropy)
    -   [BIP39 mnemonic](#bip39-mnemonic)
    -   [Individual private keys](#individual-private-keys)
    -   [Secrets](#secrets)
    -   [Seeds](#seeds)
    -   [Online Read-Only Wallet](#online-read-only-wallet)
    -   [Offline Signing Wallet](#offline-signing-wallet)
    -   [PSBT](#psbt)
-   [Overview](#overview)
    -   [Creating a wallet](#creating-a-wallet)
        -   [Verify that the wallet is spendable](#verify-that-the-wallet-is-spendable)
    -   [Sending funds](#sending-funds)
    -   [Receiving funds](#receiving-funds)
    -   [Testing backups](#testing-backups)
-   [Generating secrets](#generating-secrets)
-   [Storing seeds](#storing-seeds)
-   [Encrypting seeds at rest](#encrypting-seeds-at-rest)
    -   [Passphrase](#passphrase)
    -   [Split seeds](#split-seeds)
    -   [Multisig seeds](#multisig-seeds)
    -   [Discussion: split vs multisig seeds](#discussion-split-vs-multisig-seeds)
-   [Secure air-gapped communication](#secure-air-gapped-communication)
-   [Notes and references](#notes-and-references)
    -   [Cold storage protocols](#cold-storage-protocols)
    -   [Passphrase generation](#passphrase-generation)
    -   [Seed generation](#seed-generation)
    -   [HD wallets address derivation](#hd-wallets-address-derivation)
    -   [Air-gapped communication](#air-gapped-communication)
        -   [Audible](#audible)
        -   [Visual](#visual)
    -   [Wallets: multisig coordinators](#wallets-multisig-coordinators)
    -   [Wallets: desktop](#wallets-desktop)
    -   [Wallets: Android](#wallets-android)
    -   [Wallets: hardware](#wallets-hardware)
        -   [Trezor vs Ledger](#trezor-vs-ledger)
    -   [Wallets: utilities](#wallets-utilities)
    -   [Split HD seeds and secret sharing (SLIP39)](#split-hd-seeds-and-secret-sharing-slip39)
    -   [Split HD seeds and secret sharing (non-standard implementations)](#split-hd-seeds-and-secret-sharing-non-standard-implementations)
    -   [Multisig](#multisig)
    -   [Operation systems for cold storage](#operation-systems-for-cold-storage)
    -   [General](#general)

## Goals

-   Serve as a guide for secure storage and transacting of cryptocoins.
-   Avoid any novelty. It's basically bits and pieces from the web, organized in
    a way that I personally find clear.
-   Use simple, clear, and succinct language.
-   Be comprehensive but succinct: cover many aspects but link to other sources
    wherever possible.
-   Use methods that support multiple currencies where possible, but focus on
    Bitcoin and Ethereum (including ERC-20 tokens).

### Non-Goals

-   Cover the purchase of cryptocoins. Only secure storage and transacting are
    in scope.
-   Be friendly to novices. This guide aims to be succinct and assumes
    familiarly with concepts such as HD wallets, BIP32/39/43/44/45, Segwit, and
    Multisig wallets. If those don't mean much to you, this guide is probably
    not for you.

## Disclaimer

I wrote this guide for my own needs. I decided to share it with the community to
collect feedback and help others in a similar position, but I take no
responsibility whatsoever for any of the written content. Specifically, I am not
responsible for any loss of funds incurred by following this guide or using any
referenced software. You should always do your own research, and don't ever
trust strangers on the internet with anything important.

## Principles

-   Minimize trust in external entities, hardware, firmware, and software.
-   Maximize simplicity.
-   Follow established protocols and stick to adopted standards that have
    multiple independent implementations.
-   Avoid single points of failure.
-   Defense in depth.
-   Least privilege.
-   Compartmentalization.

## Threat model

-   Malware and attacks originating from the Internet have a very high risk. Any
    malware infection in a device that handles secrets can compromise all the
    secrets the device holds, and possibly other secrets in devices with
    physical proximity.
-   Human errors in executing protocols are highly likely.
-   Any physical location can be compromised, whether by theft or physical
    damage.
-   Physical attacks have a low risk, but defending against simple ones
    (burglars and thieves) is in scope.
-   Physical attacks by a sophisticated adversary are out of scope.
-   Social engineering is out of scope.

## Terminology

### BIP32 seed

The pseudo-random byte sequence of length 128-512 bits used in the first step of
BIP32's
[master key derivation](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#master-key-generation).
Typically, backed up using BIP39 mnemonics.

### BIP32 root key

The extended private key at the top of the BIP32 hierarchy. Usually denoted as
`m` in BIP32 derivations.

### BIP39 entropy

An initial entropy of 128-512 bits used in
[BIP39's mnemonic generation](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki#generating-the-mnemonic).
Derives the mnemonic phrase by adding a checksum to the initial entropy and then
decoding the result using a word list.

### BIP39 mnemonic

The mnemonic phrase computed in BIP39 from the BIP39 entropy. This mnemonic is
later hashed with an optional password to derive the BIP32 seed.

### Individual private keys

Private keys which are generated independently (in contrast to keys that are
generated using a scheme such as BIP32). Less flexible than
[BIP32 seeds](#bip32-seed) since they are only associated to a single "account",
but may have security advantages if BIP32 turns out to have weaknesses.

### Secrets

Any information that can help an attacker to spend funds. This includes:

-   [BIP32 seeds](#bip32-seed)
-   [Individual private keys](#individual-private-keys)
-   Any passphrases for the above, for example a passphrase for BIP39 or a
    passphrase for BIP38 encrypted private keys.
-   Public keys: knowing them makes it easier to retrieve the corresponding
    private key by exploiting a weakness in ECDSA, whereas knowing only the
    address adds another security layer: inverting a cryptographic hash.

### Seeds

Anything that can be used to derive private keys, including BIP32 seeds, BIP39
mnemonics, and individual private keys.

### Online Read-Only Wallet

A wallet that only knows public keys and/or addresses, but not any seed. This
wallet is only used for tracking funds in existing addresses and generating
addresses for receiving new coins. It must be online because tracking funds
requires the accessing the blockchain (but generating addresses can be done
offline).

### Offline Signing Wallet

A device that is completely offline (not connected to the internet, nor any
device that can be connected to the internet) that is used to sign transactions.
Can be either a hardware wallet or a general purpose computer.

When using multisig, multiple separate offline signing wallets are required for
securely sending funds, each providing a single signature.

### PSBT

A transaction format standard for partially signed Bitcoin transactions defined
in [BIP174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki).
Facilitates interoperability between different wallets that process partially
signed transactions.

## Overview

### Creating a wallet

-   Generate seeds and auxiliary secrets using one or more air gapped devices.
-   Verify the seeds derive the same master public keys (xpub/ypub/zpub) with at
    least two independent implementations.
-   Transfer master public keys (xpub/ypub/zpub) to the online read-only wallet.
-   Verify that the new wallet can receive and spend funds (see subsection
    below).
-   Back up and distribute both the secrets and master public keys to paper in
    multiple locations.

#### Verify that the wallet is spendable

-   Generate a new address `address1` in the online read-only wallet.
-   Send a **small** amount of funds from your old wallet (can be an exchange)
    to `address1`.
-   Verify that the online read-only wallet can detect that `address1` received
    funds.
-   Generate a new address `address2` in the online read-only wallet.
-   [Send](#sending-funds) the funds from `address1` to `address2`.
-   Verify that the online read-only wallet can detect that `address1` moved
    funds to `address2`.

### Sending funds

-   Generate an unsigned transaction in the online read-only wallet.
-   For each offline signing wallet (multiple wallets are used only in
    multisig):
    -   [Transfer](#secure-air-gapped-communication) the unsigned transaction to
        the offline signing wallet.
    -   Sign the transaction in the offline signing wallets (can be multiple
        wallets if using multisig)
    -   [Transfer](#secure-air-gapped-communication) the (partially) signed
        transaction to the online read-only wallet.
-   (Multisig only): Combine the partially signed transactions in the online
    read-only wallet to a final signed transaction.
-   Broadcast the transaction from the online read-only wallet.

### Receiving funds

-   Generate a new address in the online read-only wallet and provide it to the
    sender.

### Testing backups

-   Use the stored seeds to reconstruct each of the offline signing wallets and
    [verify that you can spend funds](#verify-that-the-wallet-is-spendable) from
    these wallets.

## Generating secrets

Highly unpredictable secret generation is critical for the security of
cryptographic algorithms. The following methods should provide a sufficient
level of security:

-   Software generation on an air-gapped computer. Additional requirements:
    -   The software generating the entropy for the secrets must be verified to
        use only secure sources of entropy (for example
        [`getrandom`](http://man7.org/linux/man-pages/man2/getrandom.2.html) on
        Linux).
    -   The OS must be loaded from live media (USB drive etc), be in pristine
        condition (no modifications to the upstream version), include minimal
        software other than what's required, be verified against signatures
        provided from the developers, and leave no traces (no persistence by
        default, i.e. be "amnesic").
    -   Ideally, the OS will be booted from a read only storage medium in order
        to prevent malicious modifications. To this end, DVDs (which are read
        only after burning them) or USB flash drives with write protection can
        be used.
    -   Ideally, the computer will never be connected to the internet after it
        was used to generate the secrets. This eliminates the risk that the
        generated secret was stored in persistent memory (probably due to
        already being compromised), and then transmitted to the attacker when
        connected to the internet.
-   Firmware/hardware generation on a special purpose security device such as a
    hardware wallet or an HSM.
-   Casino grade dice. Pro: simple, malware resistant, easy to verify. Con:
    requires manual input to a computer. Also, note that some secrets require
    additional computation that can't be easily done on paper (for example, the
    checksum for a BIP39 mnemonic), which reduces the malware resistance
    advantage.

It is possible to combine multiple independent sources of randomness to derive
secrets that are at least as random as the most random source (for example, by
XORing them).

## Storing seeds

-   Store the seeds in multiple physical locations in a non-electronic form with
    redundancy, for example using identical paper copies with the seed, or
    multisig seeds (see below).
    -   This should prevent most attacks, since physical access will be required
        to compromise the secrets.
    -   Redundant storage ensures that it's possible to restore the funds in
        case a subset of the storage locations are compromised (for example due
        to loss, theft, fire, etc.).
-   Use durable materials, such as the
    [durable paper](https://www.amazon.com/dp/B076JKVNWY/ref=cm_sw_r_cp_ep_dp_0f5GAbR8XZDSA)
    Glacier recommends, or a [Cryptosteel](https://cryptosteel.com/).
-   Store the seeds in a safe place, for example a vault.
-   Consider using tamper-resistant seals to be able to detect compromises.

## Encrypting seeds at rest

If you store seeds unencrypted, an attacker can steal funds by a simple physical
theft. The following subsections discuss additional layers of defense that can
be used to prevent such attacks.

### Passphrase

You can use a passphrase to protect the seeds (both for
[BIP39 mnemonics](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki#from-mnemonic-to-seed)
and for
[private keys](https://github.com/bitcoin/bips/blob/master/bip-0038.mediawiki)).
This means that you need to store the passphrase securely as well (don't forget
about backups!). Options include:

-   On paper or other non-electronic form, in a **disjoint set of physical
    locations**.
-   In secure hardware devices such as Yubikeys or HSMs.
-   In a password manager. The con of this method is that it may be impossible
    for your heirs to access the password.

For any of these methods, backups can be made by adding redundancy, for example
storing multiple paper backups, using multiple hardware devices, or storing the
password manager database in multiple devices.

### Split seeds

Instead of directly storing the seeds, you can split them to `N` shares using
Shamir's Secret Sharing (from now on SSS) and store the shares. Using SSS you
can choose a parameter `M` so that the secret can be restored by any `M` shares.
This means an attacker needs to compromise at least M different physical
locations to steal the funds. Moreover, you can loss up to `N-M+1` shares and
still be able to restore the seed.

A similar option is to use SSS to split a _passphrase_ to the seed, instead of
the seed itself. The advantage of doing this is that the passphrase is usually
smaller, so the SSS shares will also be smaller and easier to store.

See [SLIP-0039](https://github.com/satoshilabs/slips/blob/master/slip-0039.md)
for more details on this approach.

### Multisig seeds

Using a method like
[BIP45](https://github.com/bitcoin/bips/blob/master/bip-0045.mediawiki), it's
possible to generate `N` seeds so that each transaction requires signatures from
private keys derived from `M` of the seeds. The equivalents for private keys are
[BIP11](https://github.com/bitcoin/bips/blob/master/bip-0011.mediawiki) and the
newer [BIP16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki).

### Discussion: split vs multisig seeds

-   Multisig seeds are standardized and supported at the protocol level
    ([BIP45](https://github.com/bitcoin/bips/blob/master/bip-0045.mediawiki)),
    while split seeds require custom software. This means that multisig seeds
    are likely to have better software support over time. For example, they are
    already supported by a few wallets.
-   When the secrets are stored in **cold storage**, both split and multisig
    require `M/N` shares to spend funds. In both cases, the user can choose `M`
    and `N` to fit their specific security requirements.
-   Multisig enables you to maintain the property that at no point in time there
    is sufficient information to spend funds in a single wallet/place, even when
    sending funds. When you want to spend funds, you generate a transaction,
    pass it to each of the wallets for signing, and then broadcast it to the
    network. Importantly, no single wallet will have sufficient information to
    spend funds during the whole process. The upshot is that multisig seeds
    enable you to distribute trust among multiple wallets. For example, you can
    use both a Trezor and a Ledger, each having one multisig seed in a 1-of-2
    setup, and require signatures from both seeds to sign a transaction. Even if
    a severe vulnerability is discovered in one of them, it should not affect
    you. Contrast this with split seeds, where there will necessarily be some
    time period where a single wallet has enough information to spend funds.
-   Multisig seeds require `M` signatures for every transaction. Using split
    seeds requires `M` shares to reconstruct the seed, but from that point the
    reconstructed seed can be used to sign any number of transactions.
-   Multisig seeds do not yet support multiple coins in a standardized way. Note
    that BIP45 specifies derivation paths that are specific to a single coin,
    and as of 2020-12-27 it doesn't seem to be used by any popular wallet.

## Secure air-gapped communication

Signing transactions requires moving data between the offline signing wallets
and an online read-only wallet. Using USB for such a task is risky, as the USB
stack is complex and it has been exploited in the past to infect computers with
malware ([Stuxnet](https://www.wikiwand.com/en/Stuxnet),
[BadUSB](https://hackaday.com/tag/badusb/), etc.). There are two alternative
mechanisms for communication that avoid the complexity of USB and other
electronic connections:

-   Visual codes (such as QR codes)
    -   Pro: QR codes are standardized and have a large software ecosystem.
    -   Con: Large transactions may not fit in a single QR code. Multiple QR
        codes can be used to transfer an arbitrary amount of data, but that
        increases manual work and there's no standards for that.
-   Audible codes
    -   Pro: Supports large transactions more seamlessly than QR codes.
    -   Con: No standards and limited software selection.
    -   Con: It's harder to defend against wiretapping/eavesdropping than a
        camera (microphones are smaller and harder to block).

Another consideration is hardware requirements. QR codes require a
camera/scanner, while audible codes require a microphone and speakers.

See references below for implementation options.

## Notes and references

### Cold storage protocols

-   [Glacier protocol](https://glacierprotocol.org/): I reviewed the
    [design document](https://glacierprotocol.org/assets/design-doc-v0.9-beta.pdf)
    and it looked good. The main issues with it are that it doesn't use HD
    wallets (didn't have time to review it before the release), and doesn't look
    maintained since early 2018.
    -   [BitcoinHodler's fork of Glacier](https://github.com/bitcoinhodler/GlacierProtocol)
        improves many things in Glacier and seems well maintained as of
        2020-11-28.
    -   [Proof Wallet](https://github.com/hodlwave/proof-wallet) is another
        Glacier fork which adds support for HD wallets and sequential signing.
        Seems maintained as of 2020-11-28.
-   [Subzero](https://github.com/square/subzero): Square's cold storage solution
    using HSMs for storage, QR codes for hot/cold storage communication, with
    multiple signatures required. Applied specifically to Square's hardware
    setup, but the ideas are general and the documentation is clear.
-   [10x Security Bitcoin Guide](https://btcguide.github.io): Bitcoin security
    guide by Michael Flaxman advocating for multisig seeds managed by hardware
    wallets from different vendors.
    -   [How should I store my bitcoin?](https://medium.com/@michaelflaxman/how-should-i-store-my-bitcoin-43874ac208e4):
        good article by Michael Flaxman, published on 2017-09-28.
-   [Pro-Tips for Ethereum Wallet Management](https://silentcicero.gitbooks.io/pro-tips-for-ethereum-wallet-management/content/):
    Ethereum guide from February 2018.
-   [SmartCustody](https://www.smartcustody.com/): Book by Blockchain Commons.
-   https://github.com/BlockchainCommons/Gordian
-   [Advanced: Creating a Secure Wallet by Tomshwom](https://support.mycrypto.com/staying-safe/advanced-secure-wallets-by-tomshwom):
    good guide for generating keys on an air gapped computer booting Tails. The
    guide agrees that hardware wallets are a very good option, but tries to
    avoid them due to their cost.
-   [grayolson's guide for cryptocoins cold storage](https://steemit.com/cryptocurrency/@grayolson/how-to-cold-store-your-bitcoin-ethereum-altcoins-not-in-a-hardware-paper-wallet):
    advocates storing encrypted private keys in the cloud (Dropbox etc),
    encrypted with a strong Diceware password. Uses Tails on an air gapped
    computer to generate the keys. Most of the content is pretty good but I
    don't like the general strategy of storing the sensitive data online.
-   [Coinbase Vault](https://www.coinbase.com/vault): managed wallet that
    requires multiple approvals for withdrawing funds and a time delay. Claims
    that 98% of the funds are stored offline.

### Passphrase generation

-   [Reliably generating good passwords](https://lwn.net/Articles/713806/): LWN
    article from 2017-02-08.
-   [github.com/micahflee/passphraseme](https://github.com/micahflee/passphraseme)
-   [github.com/ulif/diceware](https://github.com/ulif/diceware): python script
    for generation Diceware passphrases. Supports improved word lists published
    by the EFF in 2016. implementation of the Diceware method.
-   [github.com/sethvargo/go-diceware](https://github.com/sethvargo/go-diceware):
    golang library implementing the Diceware method.
-   [github.com/sethvargo/go-password](https://github.com/sethvargo/go-password):
    golang library for generating "standard" passwords with restrictions
    (minimum number of digits, etc.).

### Seed generation

-   [github.com/iancoleman/bip39](https://github.com/iancoleman/bip39): web tool
    for generating BIP39 mnemonics or generating the private keys from an
    existing mnemonic or external entropy. Can be used offline.
-   [bc-seedtool-cli](https://github.com/BlockchainCommons/bc-seedtool-cli): CLI
    tool for generating entropy and converting external entropy to different
    formats (hex, BIP39, binary, etc.). See also the
    [comprehensive manual](https://github.com/BlockchainCommons/bc-seedtool-cli/blob/master/Docs/MANUAL.md).
-   [github.com/trezor/python-mnemonic](https://github.com/trezor/python-mnemonic):
    reference implementation for BIP39 by Trezor.
-   [github.com/tyler-smith/go-bip39](https://github.com/tyler-smith/go-bip39):
    golang implementation of BIP39.
-   [github.com/taelfrinn/Bip39-diceware](https://github.com/taelfrinn/Bip39-diceware):
    mapping from a coin flip and 4 dice rolls to BIP39 words. Can be used to
    generate the first `N-1` words in a BIP39 mnemonic, similar to the Diceware
    method for generating passwords. Note that the last word in a BIP39 mnemonic
    is a checksum and therefore cannot be randomly generated. To get the last
    word, you will have to use something like
    [seedpicker's](https://github.com/merland/seedpicker) last word calculator,
    or a script that tries all the words, similar to
    [this script](https://github.com/jonathancross/jc-docs/blob/master/BIP39_Seed_Phrase_Checksum.py)
    by Jonathan Cross.
-   [github.com/bip32JP/bip32JP.github.io](https://github.com/bip32JP/bip32JP.github.io)

### HD wallets address derivation

-   [xpub-converter](https://github.com/Casa/xpub-converter): web tool for
    converting extended public keys between different versions.
-   [go-ethereum-hdwallet](https://github.com/miguelmota/go-ethereum-hdwallet):
    golang library to compute BIP32 derivations with an emphasis on Ethereum. As
    of 2019-12-04, looks tested and maintained.
-   [bc-keytool-cli](https://github.com/BlockchainCommons/bc-keytool-cli)
-   [pywallet](https://github.com/ranaroussi/pywallet): HD wallet creator that
    supports BIP44 (but requires forking to support BIP49). As of 2019-12-04,
    has many open issues with no response from the author.

### Air-gapped communication

#### Audible

-   [amodem](https://github.com/romanz/amodem): a tool for sending data between
    computers using audio. The codebase looks high quality (well structured and
    readable code, tests, documentation, etc.) and the author is responsive.
-   [Audio MODEM](https://github.com/romanz/audiomodem-android)- Android
    implementation of amodem by the same author.

#### Visual

-   [AirGap](https://airgap.it/): OSS mobile apps for managing the secrets in a
    dedicated offline device, and then broadcasting transactions with another
    "regular" internet connected device. Communication is done with QR codes.
-   [Guide on using Electrum and QR codes](https://medium.com/@fbonomi/a-bitcoin-cold-wallet-based-on-qr-codes-e8c130b3181f)
    for air-gapped transactions.
-   [Broadcast Transaction](https://github.com/JJandJ/Broadcast-Transaction):
    Android app that supports broadcasting transactions via QR codes for BTC,
    BCH, LTC, DASH, and ZCASH. Not updated since 2018-08-22.

### Wallets: multisig coordinators

-   [Specter Desktop](https://github.com/cryptoadvance/specter-desktop): A
    desktop GUI for Bitcoin Core optimised for multisig workflows with offline
    signing wallets.
-   https://github.com/unchained-capital/caravan
-   https://github.com/sparrowwallet/sparrow
-   Electrum
-   [Junction](https://github.com/justinmoon/junction): GUI for using hardware
    wallets with Bitcoin Core. Uses HWI to interface with hardware wallets. As
    of 2020-12-25, seems to be testnest only, and unmaintained since
    October 2019.

### Wallets: desktop

-   [List from bitcoin.org](https://bitcoin.org/en/choose-your-wallet?step=5&platform=linux)
-   [Bitcoin Core](https://bitcoin.org/en/bitcoin-core/)
-   [Specter Desktop](https://github.com/cryptoadvance/specter-desktop): A
    desktop GUI for Bitcoin Core optimised for multisig workflows with offline
    signing wallets.
-   [Electrum](https://electrum.org/#home): one of the most popular bitcoin
    wallets. I'm worried about its security and privacy
    ([see this HN thread](https://news.ycombinator.com/item?id=18770577#18771112)),
    though some of the issues can be mitigated by running an Electrum Server.
    -   [electrs](https://github.com/romanz/electrs): an efficient
        re-implementation of Electrum Server in Rust. Great performance and
        codebase looks high quality.
-   [Beancounter](https://github.com/square/beancounter): utility for auditing
    the balance of Hierarchical Deterministic (HD) wallets. Supports multisig +
    segwit wallets.
-   [MyEtherWallet](https://github.com/MyEtherWallet/MyEtherWallet)
-   [MyCrypto](https://github.com/MyCryptoHQ/MyCrypto)
-   [coinbin](https://github.com/OutCast3k/coinbin/)
-   [Copay](https://github.com/bitpay/copay): TODO
-   [Wasabi](https://github.com/zkSNACKs/WalletWasabi/): privacy focused bitcoin
    wallet supporting CoinJoin. As of 2019-10-30, seems to have very active
    development.
-   [Gnosis Safe](https://gnosis-safe.io/): Ethereum wallet with support for
    multisig with Trezor and Ledger
    -   <https://github.com/gnosis/MultiSigWallet>: Older Gnosis multisig wallet
-   [gowallet](https://github.com/aiportal/gowallet): cross platform TUI wallet
    written in golang. Doesn't look maintained, and uses custom cryptography.

### Wallets: Android

-   [List from bitcoin.org](https://bitcoin.org/en/choose-your-wallet?step=5&platform=android)
    for Android
-   [BlueWallet](https://github.com/BlueWallet/BlueWallet): Android and iOS
    wallet with support for SegWit, RBF, and Lightning network.
-   [Copay](https://github.com/bitpay/copay): TODO
-   [Samourai Android Wallet](https://github.com/Samourai-Wallet/samourai-wallet-android)

### Wallets: hardware

An issue common to most hardware wallets, including Trezor and Ledger, is that
they are not truly air gapped: they use SD cards or USB to communicate with an
internet connected host. Cobo Vault is one device which does this right, and
uses QR codes to communicate with the host.

-   [List from bitcoin.org](https://bitcoin.org/en/choose-your-wallet?step=5&platform=hardware)
-   <https://diybitcoinhardware.com>: references for building a DYI hardware
    wallet
    -   https://github.com/cryptoadvance/specter-diy
    -   [Build a hardware wallet from scratch](https://github.com/stepansnigirev/workshop_advbitcoin)
-   [LetheKit](https://github.com/BlockchainCommons/bc-lethekit): DYI crypto
    device and hardware wallet from Blockchain Commons.
-   [SatoChipApplet](https://github.com/Toporin/SatochipApplet): DIY wallet
    based on a Java Card applet. Can be run on a Yubikey Neo.

#### Trezor vs Ledger

The most popular wallet vendors are [Trezor](https://trezor.io/) and
[Ledger](https://www.ledger.com/), and both support a large number of
currencies.

Trezor seems better to me since it's fully OSS (hardware, firmware, and
software), and both of the founders have been active in the Bitcoin community
for a long time and seem to have a good reputation. That said, Trezor had at
least one critical issue in the past—it
[stored the device PIN in plaintext](https://news.ycombinator.com/item?id=15589718).
To their defense, they quickly fix reported security issues, and are responsive
in general.

Ledger also had
[serious security issues](https://saleemrashid.com/2018/03/20/breaking-ledger-security-model/)
in the past, and their response wasn't great—according to linked article, their
CEO "made some comments on Reddit which were fraught with technical inaccuracy".
See the
[Interaction with Ledger](https://saleemrashid.com/2018/03/20/breaking-ledger-security-model/#interaction-with-ledger)
section for more details.

### Wallets: utilities

-   <https://walletsrecovery.org>: List of wallets with their supported
    derivation paths and other features, aimed to help wallet users understand
    the interoperability between wallets.
-   [HD Wallet Scanner](https://github.com/alexk111/HD-Wallet-Scanner): Node.js
    tool to find used addresses in HD wallets, bypassing the gap limit of 20
    specified in BIP32.
-   [HWI](https://github.com/bitcoin-core/HWI): python library and CLI for
    interacting with hardware wallets in a unified way, compatible with multiple
    models.

### Split HD seeds and secret sharing (SLIP39)

-   [python-shamir-mnemonic](https://github.com/trezor/python-shamir-mnemonic):
    reference implementation of
    [SLIP-0039](https://github.com/satoshilabs/slips/blob/master/slip-0039.md):
    Shamir's Secret-Sharing for Mnemonic Codes.
-   [github.com/iancoleman/slip39](https://github.com/iancoleman/slip39)
-   [Hermit](https://github.com/unchained-capital/hermit): CLI wallet
    implementing [split seeds](#split-seeds) following
    [SLIP-0039](https://github.com/satoshilabs/slips/blob/master/slip-0039.md).
-   [bc-slip39](https://github.com/BlockchainCommons/bc-slip39): SLIP-39
    implementation in C from Blockchain Commons.
-   [slip39-rust](https://github.com/Internet-of-People/slip39-rust): SLIP-39
    implementation in Rust.

### Split HD seeds and secret sharing (non-standard implementations)

-   [bc-shamir](https://github.com/BlockchainCommons/bc-shamir)
-   https://github.com/dsprenkels/sss
-   https://github.com/SSSaaS/sssa-golang
-   [shamir39](https://github.com/iancoleman/shamir39)
-   [Guide on using Shamir's Secret Sharing](https://medium.com/@markstar/backup-your-trezor-ledger-using-shamirs-secret-sharing-972e98101839)
    to back up the BIP39 seed. Uses a non-standardized SSS implementation, so
    SLIP-39 should be preferred.
-   [github.com/grempe/secrets.js](https://github.com/grempe/secrets.js): SSS
    implementation in JavaScript. Reports that it successfully passed a security
    audit. Latest real activity from September 2019.
-   [github.com/shea256/secret-sharing](https://github.com/shea256/secret-sharing):
    SSS implementation in Python. Doesn't look maintained since early 2016.
-   <http://point-at-infinity.org/ssss/>: SSS implementation from 2006. Looks
    unmaintained.

### Multisig

-   [Multisig hardware wallets guide](https://saleemrashid.com/2018/01/27/hardware-wallet-electrum-multisig/):
    Shows how to use multiple hardware wallets with multisig using Electrum, so
    that M/N wallets are required for signing transactions.
-   [Ethereum multisig overview](https://medium.com/mycrypto/introduction-to-multisig-contracts-33d5b25134b2)
    from 2020-01-16.
-   [firma](https://github.com/RCasatta/firma): PSBT signer.
-   https://github.com/benthecarman/PSBT-Toolkit
-   [linux-psbt-signer](https://github.com/justinmoon/linux-psbt-signer): PSBT
    signer.

### Operation systems for cold storage

_Note_: "Live OS" means that the OS is designed to be run on any computer by
booting it from a USB stick or DVD.

-   [Tails](https://tails.boum.org/): Live Linux distro focused on privacy and
    anonymity and designed to leave no trace on the computers it's used on. All
    internet connections are forced to go through Tor. Bundled with Electrum.
    Seems to be the best maintained security/privacy focused OS based on git
    activity.
    -   <https://github.com/PulpCattel/Tails-BitcoinCore-Wasabi>: A guide for
        using Tails, Bitcoin core, and Wasabi to implement an offline wallet
    -   <https://github.com/SovereignNode/tails-cold-storage>: A guide on
        creating Bitcoin cold storage using Tails.
-   [AirGap Vault Distribution](https://github.com/airgap-it/airgap-distro):
    Debian-based Linux distro based on BitKey. Looks unmaintained, but the code
    is very simple and can probably be updated easily.
-   [BitKey](https://github.com/bitkey/bitkey): Live Linux distro with built-in
    software for processing cryptocoins such as wallets, password managers, QR
    code generator, etc. Doesn't look maintained since mid 2018.
-   [Qubes OS](https://www.qubes-os.org/): OS architecture that heavily uses
    compartmentalization to improve security. Usually runs a Debian or Redhat
    distro. Has stricter
    [hardware requirements](https://www.qubes-os.org/doc/system-requirements/)
    than a regular OS.
    -   [qubes-whonix-bitcoin](https://github.com/qubenix/qubes-whonix-bitcoin):
        guide for using Qubes and Whonix to run Electrum in an offline VM,
        connected to a separate VM serving an Electrum personal server.
-   [Whonix](https://www.whonix.org/): TODO.
-   [Kodachi](https://github.com/WMAL/kodachi): Live Linux distro similar to
    Tails: amnesic by default (no data persistence) and uses Tor to route all
    traffic. Doesn't look maintained based on Github activity.

### General

-   [Blockstream.info](https://blockstream.info/) is a good open source block
    explorer.
-   https://github.com/BlockchainCommons/Research
-   Reusing an address for withdrawals introduces risks and should be avoided.
    See details in
    [Glacier's design doc](https://glacierprotocol.org/assets/design-doc-v0.9-beta.pdf),
    section "Address Reuse and HD Wallets", page 20.
-   When typing passwords, you can increase the security by doing some of the
    typing with a virtual keyboard, which is less susceptible to hardware key
    loggers and some audio side channel attacks.
