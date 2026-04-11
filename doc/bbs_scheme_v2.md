# BBS DID Scheme - Holographic Overview

[BBS Signature Scheme (Chinese)](https://klizz.top/sqxx/BBS.html)

## System Architecture

- ***User***: The user who wants to obtain a credential (passport) to access a DApp.
- ***Issuer***: An on-chain contract deployed by the DApp on the origin chain, which can mint a soulbound token (ERC-5114 / ERC-5484, revocable by the Issuer) to the User on the destination chain.
- ***Signer***: An off-chain Trusted Third Party (TTP) service provider that verifies user information and issues BBS signatures.
- ***Verifier***: A contract deployed by the DApp/Signer that verifies the NIZK proof and triggers a callback to the Issuer contract to mint the token. Can be deployed on different chains to minimize verification costs.(Reactive Network)

## Prelude

Message coding definition：

| Message ID | Message contents |
| --- | --- |
| $m_0, H_0$ | Constrain1, e.g., "Is_Adult=1" |
| $m_1, H_1$ | Constrain2, e.g., "keccak(nationality)=keccak("CN")" |
| $m_2, H_2$ | Constrain3, ... |
| $m_3, H_3$ | Constrain4, ... |
| $m_4, H_4$ | Expiration Height |
| $m_{\gamma}, H_{\gamma}$ | Blind factor |
| $m_{\text{null}}, H_{\text{null}}$ | Nullifier commitment |

## 1. Init

+ ***Issuer*** generates unique $H_{\text{null}}$ and encodes it in the contract.

+ ***Signer*** runs $KeyGen$ to get private key $x$ and publishes the public key $X$.

+ ***Signer*** or *Issuer* deploys the *Verifier* contract and embeds $H_{\text{null}}$ and $X$ into the contract.

> [!NOTE]
>
> Current status  
>
> + ***Public***: $H_{\text{null}}, X, \vec{\mathbb{m}}$
>   + ***Signer***: $x$

## 2. User apply

- ***User***
  + Use `view` function to read $H_{\text{null}}$ and the requirements $(\vec{\mathbb{m}})$ of DApp on chain.
  
  + Sample $m_{\text{null}}, m_{\gamma}, \lambda \leftarrow \mathbb{Z}_p$, compute:
    - Nullifier hash: $\mathcal{N} = \text{Hash}(0x\text{UserAddress} \,||\, m_{\text{null}})$
    - Nullifier commitment: $\mathcal{N}' = m_{\text{null}} \cdot H_{\text{null}} + m_{\gamma} \cdot H_{\gamma}$
    - Bind the commit: $\mathcal{C}_2$ = $\lambda \cdot (m_{\text{null}} H_{\text{null}} + m_{\gamma} H_{\gamma})$
  
  + Send $\mathcal{N}$ to *Verifier* Contract to stash it on chain (`KeyExist[mapping(uint=>bool)][N]] = true`) by using a new address $0x\text{RelayerAddr}$.
  
  + Aggregate the constraints $\vec{\mathbb{m}} = \{m_0, m_1, m_2, \dots\}$.
  
  + Send $(\vec{\mathbb{m}}, \mathcal{C}_2)$ to Signer.
  
  > [!NOTE]
  >
  > Current status
  >
  > + ***Public***: $H_{\text{null}}, X, \vec{\mathbb{m}}$
  >   + ***Verifier***: $\langle \mathcal{N}, 0x\text{RelayerAddr} \rangle$  
  > + ***Signer***: $x$, $\langle \text{UserID}, \vec{\mathbb{m}}, \mathcal{C}_2 \rangle$
  > + ***User***: $0x\text{UserAddr}, m_{\text{null}}, m_{\gamma}, \lambda$
  
  
  

## 3. Signer gen signature

+ ***Signer***
  + Check whether the user's info corresponds to the constraints in $\vec{\mathbb{m}}$. If not, refuse to sign.
  + Compute 1st commitment:
    $$
    \mathcal{C}_1 = G_1 + \sum_{i=0}^{4} m_i \cdot H_i \in \mathbb{G}_1
    $$
  + Generate 1st signature $\sigma' = (A_1, e)$ where:
    $$
    e \xleftarrow{\$} \mathbb{Z}_p, \quad A = \frac{1}{x + e} \cdot \mathcal{C}_1
    $$ 
  + Use same $e$ to generate 2nd signature $\sigma' = (A_2', e)$ where:
    $$
    A' = \frac{1}{x + e} \cdot \mathcal{C}_2
    $$
  + Send signature $\sigma' = (A_1, A_2', e)$ back to User.

> [!NOTE]
>
> Current status
>
> + ***Public***: $H_{\text{null}}, X, \vec{\mathbb{m}}$
>   + ***Verifier***: $\langle \mathcal{N}, 0x\text{RelayerAddr} \rangle$  
> + ***Signer***: $x$, $\langle \text{UserID}, \mathcal{N}', \sigma' \rangle$
> + ***User***: $0x\text{UserAddr}, m_{\text{null}}, m_{\gamma}$, $\sigma'$



## 4. User gen NIZK proof

+ ***User***
  + Unnlind the $A_2'$:
    $$
    A_2 = A_2' \cdot \lambda^{-1} = \frac{m_{\text{null}} \cdot H_{\text{null}} + m_{\gamma} \cdot H_{\gamma}}{x+e}
    $$
  + Reconstruct the signature $\sigma = (A, e)$:
  $$
    \sigma = (A, e) = (A_1+A_2, e)
  $$
  + Generate **NIZK proof** $\pi$ to prove knowledge of a valid signature $\sigma$ for messages $(\vec{\mathbb{m}}, m_{\gamma}, m_{\text{null}})$ while hiding $(m_{\gamma}, A, e)$.
  
    $$
    \pi = \left( \overline{A},\ \overline{B},\ U,\ s,\ t,\ u_{\gamma} \right)
    $$

    Let $J = \{0, 1, 2, 3, 4, \text{null}\}$ (public message indices), while $I = \{\gamma\}$ is the hidden message.
    
    $$
    \begin{aligned}
      \mathcal{C} & = \mathcal{C}_J + \mathcal{C}_I \\
      & = G_1 + \sum_{l \in \{1,2,3,4,\cdots\}} m_l \cdot H_l + m_{\text{null}} \cdot H_{\text{null}} + m_{\gamma} \cdot H_{\gamma} 
    \end{aligned}
    $$ 

    where:
    - $\overline{A} = r \cdot A$ (randomized signature point)
    - $\overline{B} = r \cdot \mathcal{C} - r \cdot e \cdot A$
    - $\mathcal{C}_J =G_1 + \sum_{j \in J} m_j \cdot H_j$
    - $U = \alpha \cdot C_J + \beta \cdot \overline{A} + \delta_{\gamma} \cdot H_{\gamma}$ 
    - $c = H(\mathsf{ctx} \,||\, \vec{m}_J \,||\, \overline{A} \,||\, \overline{B} \,||\, U)$ (FS transform)
    - $s = \alpha + r \cdot c$, $t = \beta - e \cdot c$ (responses for $r, e$)
    - $u_{\gamma} = \delta_{\gamma} + r \cdot m_{\gamma} \cdot c$

  
  + Send $(\pi, m_{\text{null}}, 0x\text{UserAddr})$ to ***Verifier***.

> [!NOTE]
>
> Current status
>
> + ***Public***: $H_{\text{null}}, X, \vec{\mathbb{m}}$
>   + ***Verifier***: $\langle \mathcal{N}, 0x\text{RelayerAddr}, 0x\text{UserAddr}, m_{\text{null}}, \pi \rangle$
> + ***Signer***: $x$, $\langle \text{UserID}, \mathcal{N}', \sigma' \rangle$
> + ***User***: $\langle 0x\text{UserAddr}, m_{\text{null}} , \sigma',\pi \rangle$

## 5. Verify the Signature and mint the token

+ ***Verifier*** 
  + Compute public part commitment (pre-computed since constraints are hard-coded):
    $$
    \mathcal{C}_J = G_1 + \sum_{j \in J} m_j \cdot H_j
    $$
    where $J = \{0, 1, 2, 3, 4, \text{null}\}$ (public message indices).
  
  + **Verify nullifier pre-commitment**:
    - Compute $\mathcal{N} = \text{Hash}(0x\text{UserAddr} \,||\, m_{\text{null}})$
    - Require `KeyExist[N] == true` (check pre-committed in Step 2)
    - Require `Used[N] == false` (prevent double-spending)
  
  + **Verify the NIZK proof** $\pi = (\overline{A}, \overline{B}, U, s, t, u_{\gamma})$:
    
    1. Recompute the Fiat-Shamir challenge (note: $m_{\text{null}}$ is included in $\vec{m}_J$ as it's revealed):
       $$
       c' = H(\mathsf{ctx} \,||\, \vec{m}_J \,||\, \overline{A} \,||\, \overline{B} \,||\, U)
       $$
    
    2. **Pairing check** (verify randomized signature is valid):
       $$
       e(\overline{A}, X) \stackrel{?}{=} e(\overline{B}, G_2)
       $$
    
    3. **Homomorphic check** (verify correct representation of $\overline{B}$ with hidden $m_{\gamma}$):
       $$
       U + c' \cdot \overline{B} \stackrel{?}{=} s \cdot C_J + t \cdot \overline{A} + u_{\gamma} \cdot H_{\gamma}
       $$
    
    If any check fails, revert.
  
  + **Mark nullifier as used**:
    - Set `Used[N] = true`
  
  + Call the Issuer's function `mint(timestamp, 0xUserAddr)` to issue the soulbound token to `0xUserAddr`.

## 6. Summary

### Features

- [x] **Unforgeability**: Only the Signer with secret key $x$ can generate valid signatures.
- [x] **Unlinkability**: The ***Signer*** cannot track where the User uses the signature because:
  - The NIZK proof is randomized ($\overline{A} = r \cdot A$ with fresh $r$ each time)
  - The Signer only sees $\mathcal{N}' = m_{\text{null}} H_{\text{null}} + m_{\gamma}H_{\gamma}$, even though $m_{\text{null}}, 0x\text{UserAddr}$ is revealed at spend time and holds the realtion of $(m_{\text{null}}, H_{\text{null}}, H_{\gamma})$ simutanuesly, the ***Signer*** should iterate every possible $m_{\gamma}$ and $\lambda$ to track the ***User***.
- [x] **Single-use**: The ***User*** cannot use the same signature multiple times because:
  - $\mathcal{N} = \text{Hash}(0x\text{UserAddr} \,||\, m_{\text{null}})$ is pre-committed before signing
  - Verifier checks and marks $\mathcal{N}$ as used atomically
- [x] **Privacy**: 
  - $m_{\gamma}$ (blind factor) is hidden via zero-knowledge in the proof
  - Signature $(A, e)$ is hidden via randomization
- [x] **Domain separation**: The $\mathsf{ctx}$ parameter in Fiat-Shamir transform prevents proof replay across different DApps.
- [x] **Expiration**: The token could be burn after a specified block height (encoded in $m_4$, e.g.) to limit the validity period of the credential, or a real-world time by aquiring the current time from a external oracle.

### Extensions

- [x] **Multiple signatures**: ...
- [x] **The length of the Messages**: the $\ell$ could be freely extended by adding more $H_i$ in the public parameters and encoding more constraints in the signature.
- [x] **Support Timed Encryption?**: When the ***User*** gen $m_{\gamma}$, the ***User*** could use *RSW* to encrypt $m_{\gamma}$ to *feature* and gen a zk-proof, so the ***Verifier*** could aquire $m_{\gamma}$ after some time delay and eventually link the *VC* to the exact ***User***.
- [x] **Support zkvm?**: The ***User*** could utilize the zkvm network to compress the proof into only one groth16 proof and minimize the on-chain verification cost.  

### Remaining Issues

- [ ] **Signature malleability**: The basic BBS signature $\sigma = (A, e)$ is susceptible to forgery $\sigma' = (A^2, e)$ for commitment $C^2$. To solve this, we should consider using BBS+.
- [ ] **Nullifier waste**: User commits $\mathcal{N}$ before receiving signature. If Signer refuses, the nullifier is wasted.
