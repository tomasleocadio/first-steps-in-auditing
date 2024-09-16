---
title: Protocol Audit Report
author: Tomás Leocádio
date: September 16, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
  - \graphicspath{{./audit-data}}
---
\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries Protocol Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape TT\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Tomás Leocádio]()
Lead Auditors: 
- Tomás Leocádio

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Storing the password on-chain makes it visible to anyone, and no longer private](#h-1-storing-the-password-on-chain-makes-it-visible-to-anyone-and-no-longer-private)
    - [\[H-2\] `PasswordStore::setPassword` has no access controls, meaning a non-owner could change the password](#h-2-passwordstoresetpassword-has-no-access-controls-meaning-a-non-owner-could-change-the-password)
  - [Informational](#informational)
    - [\[I-1\] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing natspec to be incorrect](#i-1-the-passwordstoregetpassword-natspec-indicates-a-parameter-that-doesnt-exist-causing-natspec-to-be-incorrect)

# Protocol Summary

Protocol does X, Y, Z

# Disclaimer

The Tomás Leocádio team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

\begin{table}[h!]
\renewcommand{\arraystretch}{1.2} % Increase space between rows
\begin{tabular}{|l|l|l|l|l|}
\hline
\multicolumn{2}{|c|}{} & \textbf{Impact} & & \\
\hline
\textbf{Likelihood} & & High & Medium & Low \\
\hline
High & & H & H/M & M \\
\hline
Medium & & H/M & M & M/L \\
\hline
Low & & M & M/L & L \\
\hline
\end{tabular}
\caption{Risk Classification Matrix}
\end{table}

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

**The findings described in this document correspond to the following commit hash:**

Commit hash:
```
(commit hash here)
```
## Scope 

```
./src/
#-- PasswordStore.sol
```

## Roles

- Owner: The user who can set the password and read the password.
- Outsides: No one else should be able to set or read the password.

# Executive Summary

*Add some notes about how the audit went, types of things you found, etc.*

*We spent X hours with Z editors and used Y tools, etc.*

## Issues found

\begin{table}[h!]
\begin{tabular}{|c|c|}
\hline
\textbf{Severity} & \textbf{Number of Issues Found} \\
\hline
High & 2 \\
Medium & 0 \\
Low & 0 \\
Info & 1 \\
\hline
\end{tabular}
\caption{Summary of Issues Found}
\end{table}

# Findings

## High

### [H-1] Storing the password on-chain makes it visible to anyone, and no longer private

**Description:** All data stored on-chain it's visible to anyone, and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be a private variable and only accessed through the `PasswordStore::getPassword` function, wich is intended to be only called by the owner of the contract.

We show one such method of reading any data off chain below.

**Impact:** Anyone can read the private password, severly breaking the functionality of the protocol.

**Proof of Concept:** (Proof of Code)

The below test case shows anyone can read the password directly from the blockchain.

1. Create a locally running chain 

```bash
make anvil
```

2. Deploy the contract to the chain

```bash
make deploy
```

3. Run the storage tool

We use `1` because that's the storage slot of `s_password` in the contract.

```bash
cast storage 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9 1 --rpc-url http://127.0.0.1:8545
```

You'll get an output that looks like this:

`0x6d7950617373776f726400000000000000000000000000000000000000000014`

You can parse that hex to a string with:
```bash
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

And returns the password:

`myPassword`

**Recommended Mitigation:** Due to this. the overall architecture of the contract should be rethought. One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. However, you'd also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with the password that decrypts your password.

### [H-2] `PasswordStore::setPassword` has no access controls, meaning a non-owner could change the password

**Description:** The `PasswordStore::setPassword` function is set to be an `external` function, however, the natspec of the function and overall purpose of the smart contract is that `This function allows only the owner to set a new password.`

```javascript
    function setPassword(string memory newPassword) external {
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** Anyone can set/change the password of the contract, severly breaking the contract intended functionality..

**Proof of Concept:** Add the following to the `Password.t.sol` test file.

<details>
<summary> Code </summary>

```javascript
    function test_anyone_can_set_password(address randomAddress) public {
        vm.assume(randomAddress != owner);
        vm.prank(randomAddress);
        string memory expectedPassword = "my new password";
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(expectedPassword, actualPassword);
    }
```

</details>

**Recommended Mitigation:** Add an access control conditional to the `setPassword` function.

```javascript
    if(msg.sender != s_owner) {
        revert PasswordStore__NotOwner();
    }
```

## Informational

### [I-1] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing natspec to be incorrect

**Description:** 

```javascript
    /*
     * @notice This allows only the owner to retrieve the password.
     * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {
```

The `PasswordStore::getPassword` function signature is `getPassword()` wich the natspec day it should be `getPassword(string)`.

**Impact:** The natspec is incorrect

**Recommended Mitigation:** Remove the incorrect natspec line.

```diff
- * @param newPassword The new password to set.
```
