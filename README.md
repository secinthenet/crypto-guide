# Secure cryptocurrency management

> ⚠️ **NOTE**: WIP.

## Goals

-   Serve as a guide for secure storage and transacting of cryptocurrencies.
-   Avoid any novelty. It's basically bits and pieces from the web, organized in
    a way that I personally find clear.
-   Use simple, clear, and succinct language.
-   Be comprehensive but succinct: cover many aspects but link to other sources
    wherever possible.
-   Use methods that support multiple currencies where possible, but focus on
    Bitcoin and Ethereum.

## Non-Goals

-   Cover the purchase of cryptocurrencies. Only secure storage and transacting
    are in scope.
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
-   Follow established protocols and stick to standards.
-   Avoid single points of failure.
-   Defense in depth.

## Threat model

-   Malware and attacks originating from the Internet have a very high risk. Any
    malware infection in a device that handles secrets is assumed to fully
    compromise the secrets.
-   Physical attacks have a low risk, but defending against simple ones
    (burglars and thieves) is in scope.
-   Physical attacks by a sophisticated adversary are out of scope.
-   Social engineering is out of scope.

### Secrets

In this doc, we refer to _secrets_ as information that can help an attacker to
spend funds. This includes:

-   BIP32 seeds, stored as BIP39 mnemonics. These can be used to generate an
    infinite number of private keys for multiple currencies.
-   Individual private keys which are generated independently (not from a BIP32
    seed). They are less flexible than BIP32 seeds since they are only
    associated to a single "account", but may have security advantages if BIP32
    turns out to have weaknesses.
-   Any passphrases for the above, for example an "additional word" for BIP39 or
    a passphrase for BIP38 encrypted private keys.
-   Public keys: knowing them makes it easier to retrieve the corresponding
    private key by exploiting a weakness in ECDSA, whereas knowing only the
    address adds another security layer: inverting a cryptographic hash.

From now on, "seeds" may refer to raw BIP32 seeds, BIP39 mnemonics, or
individual private keys.

## Overview

### Creating a wallet

-   Generate seeds and auxiliary secrets using an air gapped device.
-   Transfer master public address (xpub/ypub/zpub) to a hot wallet.
-   Back up and distribute the secrets to cold/offline storage in multiple
    locations.

### Spending funds

-   Generate an unsigned transaction in a hot wallet.
-   [Transfer](#secure-air-gapped-communication) the unsigned transaction to
    cold wallet.
-   Sign the transaction in the cold wallet.
-   [Transfer](#secure-air-gapped-communication) signed transaction to the hot
    wallet.
-   Broadcast transaction from the hot wallet.

### Receiving funds

-   Generate new address in a hot wallet and provide it to the sender.

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
    -   The OS should be loaded from a DVD instead of a USB flash drive since
        the former is more secure against modifications after the initial write.
-   Firmware/hardware generation on a special purpose security device such as a
    hardware wallet or an HSM.
-   Casino grade dice. Pro: simple, malware resistant, easy to verify. Con:
    requires manual input to a computer. Also, note that some secrets require
    additional computation that can't be easily done on paper (for example, the
    checksum for a BIP39 mnemonic), which reduce the malware resistance
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
-   Consider using tamper-resistant seals.

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
-   In a password manager which is backed up to multiple devices.
-   In multiple secure hardware devices such as Yubikeys or HSMs.

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

### Multisig seeds

Using ([BIP45](https://github.com/bitcoin/bips/blob/master/bip-0045.mediawiki)),
it's possible to generate `N` seeds so that each transaction requires signatures
from private keys derived from `M` of the seeds. The equivalents for private
keys are [BIP11](https://github.com/bitcoin/bips/blob/master/bip-0011.mediawiki)
and the newer
[BIP16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki).

### Discussion: split vs multisig seeds

-   Multisig seeds are standardized and supported at the protocol level
    ([BIP45](https://github.com/bitcoin/bips/blob/master/bip-0045.mediawiki)),
    while split seeds require custom software. This means that multisig seeds
    are likely to have better software support over time. For example, they are
    already supported by wallets.
-   When the secrets are stored in **cold storage**, both split and multisig
    require `M/N` shares to spend funds. In both cases, the user can choose `M`
    and `N` to fit their specific security requirements.
-   Multisig enables you to maintain the property that at no point in time there
    is sufficient information to spend funds in a single wallet/place, even when
    spending funds. When you want to spend funds, you generate a transaction,
    pass it to each of the wallets for signing, and then broadcast it to the
    network. Importantly, any single wallet will not have sufficient information
    to spend funds during the whole process. The upshot is that multisig seeds
    enable you to distribute trust among multiple wallets. For example, you can
    use both a Trezor and a Ledger, each having one multisig seed in a 1-of-2
    setup, and require signatures from both seeds to sign a transaction. Even if
    a severe vulnerability is discovered in one of them, it should not affect
    you. Contrast this with split seeds, where there will necessarily be some
    time period where a single wallet has enough information to spend funds.
-   Multisig seeds require `M` signatures for every transaction. Using split
    seeds requires `M` shares to reconstruct the seed, but from that point the
    reconstructed seed can be used to sign any number of transactions.

## Secure air-gapped communication

Signing transactions requires moving data between the cold and hot wallets.
Using USB for such a task is risky, as the USB stack is complex and it has been
exploited in the past to infect computers with malware
([Stuxnet](https://www.wikiwand.com/en/Stuxnet),
[BadUSB](https://hackaday.com/tag/badusb/), etc.). There are two alternative
mechanisms for communication that avoid the complexity of USB and other
electronic connections:

-   Visual codes (such as QR codes)
    -   Pros: QR codes have a large software ecosystem
    -   Cons: May not support transfer large transactions (unless using multiple
        codes which adds complexity), requires a camera/scanner
-   Audible codes
    -   Pros: Supports large transactions, only requires a microphone and
        speakers which are more common than a camera
    -   Cons: Limited software selection

See references below for implementation options.

## Notes and references

### Passphrase generation

-   [Reliably generating good passwords](https://lwn.net/Articles/713806/): LWN
    article from 2017-02-08.
-   [github.com/micahflee/passphraseme](https://github.com/micahflee/passphraseme):
-   [github.com/ulif/diceware](https://github.com/ulif/diceware): python
    implementation of the diceware method.
-   [github.com/sethvargo/go-diceware](https://github.com/sethvargo/go-diceware):
    golang library implementing the diceware method.
-   [github.com/sethvargo/go-password](https://github.com/sethvargo/go-password):
    golang library for generating "standard" passwords with restrictions
    (minimum number of digits, etc.).

### Seed generation

-   [github.com/iancoleman/bip39](https://github.com/iancoleman/bip39): can be
    used for generating BIP39 mnemonics or generating the private keys from an
    existing mnemonic.
-   [github.com/trezor/python-mnemonic](https://github.com/trezor/python-mnemonic)
-   [github.com/tyler-smith/go-bip39](https://github.com/tyler-smith/go-bip39)

### Air-gapped communication

#### Audible

-   [amodem](https://github.com/romanz/amodem): a tool for sending data between
    computers using audio. The codebase looks high quality (well structured and
    readable code, tests, documentation, etc.) and the author is responsive.
-   [Audio MODEM](https://github.com/romanz/audiomodem-android)- Android
    implementation of amodem by the same author.

#### Visual

-   [air-gap](https://air-gap.it/): OSS mobile apps for managing the secrets in
    a dedicated offline mobile phone, and then broadcasting transactions with
    another "regular" internet connected mobile phone. Communication is done
    with QR codes.
-   [Guide on using Electrum and QR codes](https://medium.com/@fbonomi/a-bitcoin-cold-wallet-based-on-qr-codes-e8c130b3181f)
    for air-gapped transactions.

### Software wallets

-   [Electrum](https://electrum.org/#home): one of the most popular bitcoin
    wallets. I'm worried about its security and privacy
    ([see this HN thread](https://news.ycombinator.com/item?id=18770577#18771112)),
    though some of the issues can be mitigated by running an Electrum Server.
    -   [electrs](https://github.com/romanz/electrs): An efficient
        re-implementation of Electrum Server in Rust. Great performance and
        codebase looks high quality.
-   [Bitcoin Core](https://bitcoin.org/en/bitcoin-core/)
-   [MyEtherWallet](https://github.com/MyEtherWallet/MyEtherWallet)
-   [MyCrypto](https://github.com/MyCryptoHQ/MyCrypto)

### Mobile wallets

-   [Samourai Android Wallet](https://github.com/Samourai-Wallet/samourai-wallet-android)

### Hardware wallets

The most popular wallet vendors are [Trezor](https://trezor.io/) and
[Ledger](https://www.ledger.com/), and both support a large number of
currencies.

Trezor seems better to me since it's fully OSS (hardware, firmware, and
software), and both of the founders have been active in the Bitcoin community
for a long time and seem to have a good reputation. That said, Ledger had at
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

#### Other references

-   [Bitcoin Multisig Hardware Wallet Comparison](https://bitcoin-hardware-wallet.github.io/):
    notably, the Trezor One is missing.
-   [HWI](https://github.com/bitcoin-core/HWI): python library and CLI for
    interacting with hardware wallets, compatible with multiple models.

### Cold storage methods

-   [Glacier protocol](https://glacierprotocol.org/): I reviewed the
    [design document](https://glacierprotocol.org/assets/design-doc-v0.9-beta.pdf)
    and it looked good. The main issues with it are that it doesn't use HD
    wallets (didn't have time to review it before the release), and doesn't look
    maintained since early 2018.
-   [Subzero](https://github.com/square/subzero): Square's cold storage solution
    using HSMs for storage, QR codes for hot/cold storage communication, with
    multiple signatures required. Applied specifically to Square's hardware
    setup, but the ideas are general and the documentation is clear.
-   [How should I store my bitcoin?](https://medium.com/@michaelflaxman/how-should-i-store-my-bitcoin-43874ac208e4):
    good article from Michael Flaxman, published on 2017-09-28.
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

### Multisig and splitting secrets

-   [Guide on using Shamir's Secret Sharing](https://medium.com/@markstar/backup-your-trezor-ledger-using-shamirs-secret-sharing-972e98101839)
    to back up the BIP39 seed.
-   [python-shamir-mnemonic](https://github.com/trezor/python-shamir-mnemonic):
    reference implementation of SLIP-0039: Shamir's Secret-Sharing for Mnemonic
    Codes.
-   [Multisig hardware wallets guide](https://saleemrashid.com/2018/01/27/hardware-wallet-electrum-multisig/):
    Shows how to use multiple hardware wallets with multisig using Electrum, so
    that M/N wallets are required for signing transactions.

### General

-   Reusing an address for withdrawals introduces risks and should be avoided.
    See details in
    [Glacier's design doc](https://glacierprotocol.org/assets/design-doc-v0.9-beta.pdf),
    section "Address Reuse and HD Wallets", page 20.
-   As of 2019-09-10, it seems that using multisig is still not easy with BTC.
    bitcoin-core added support for
    [BIP-174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)
    which defines a format for partially signed transactions, which can be
    generated and passed to wallets to collect the required signatures, but it's
    not well supported by wallets yet.