# Holographic Overview of BBS scheme

## System architecture

+ ***User***: the user who wants to apply a passport to access some dapp.

+ ***Issuer***: deployed by Dapp at origin chain, a the on-chain contract, which could mint a soulbind token to user ( ERC-5114 & ERC-5484(could be withdrew by the issuer)) - (destination chain)
+ ***Signer***: a off-chain Trust Third Party service provider, checking the info of the user and then issuing  the signature
+ ***Verifier***: Deployed by the DApp/Signer, the Verifier triggers a callback event to call the original contract function and mint the token after verifying the signature. Could be deployed at different (origin chain) to minimize the cost of verification.

## Prelude

Message coding definition：

| Message ID | Message contents |
| --- | --- |
| $m0\|H_0$ | Constrain1, e.g., "Is_Adult=1" |
| $m1\|H_1$ | Constrain2, e.g., "keccak(nationality)=keccak("CN") |
| $m2\|H_2$ | Constrain3, ... |
| $m3\|H_3$ | Constrain4, ... |
| $m4\|H_4$ | Expiration Height |
| $m_{\gamma}||H_{\gamma}$ |  Blind factor |
| $m_{nullifier}\|H_{nullifier}$ | privekey commitment |

+ the $H_{null}$ should be hard coded at the issuer's contract

## 1. Init

+ ***Issuer*** gen unique $H_{nullifier}$ and encode it in the contract.

+ ***Signer*** runs $KeyGen$ to get private key $x$ and publish the pub key $X$.

+ ***Signer*** or *Issuer* deploys the *Verifier* contract and embed the $H_{null} \; \& \; X$  into the contract.

> [!NOTE]
>
> Current status  
>
> + ***Public***:: $H_{nullifier}, X$ , $\vec{\mathbb{m}}$
> + ***SIgner***: $x$



## 2. User apply

- ***User***
  + Use `view` function to read the $H_{null}$ and the requirements($\vec{\mathbf{m}}$) of Dapp on chain.
  
  + Sample $m_{nuliifier}, m_{\gamma} \in \mathbb{Z}$, calc $\mathcal{N} = Hash(0xUserAddress || m_{nullifier})$ and then send it to *Verifier* Contract to stash it on chain (*Keyexist=mapping(uint=>bool)[N]=true*) by using a new address $0xRelayerAddr$.
  
  + Aggregate the constrains $\vec{\mathbf{m}}=\{m_0, m_1, m_2, \dots\}$.
  
  + Send $(\mathbf{m}, \mathcal{N}'=m_{nullifier} \cdot H_{nullifier} + m_{\gamma} H_{\gamma})$ to signer.
  
  > [!NOTE]
  >
  > Current stataus
  >
  > + ***Public***:: $H_{nullifier}, X, \vec{\mathbf{m}}$
  > + ***Verifier***: $<\mathcal{N}, 0xRelayerAddr>$  
  > + ***SIgner***: $x$, <$UserID$, $\mathcal{N}'$>
  > + ***User***: $<0xUserAddr, nulifier$>, $m_{\gamma}$ 
  
  
  

## 3. Signer gen signature

+ Signer
  + Check whether the user's info correspond to the constraints in $\mathbf{m}$, if not, refuse to sign.
  + Calc full commitment $\mathcal{C} = G_1 + m_0H_0 + \cdots + m_{\ell}+ \mathcal{N}$
  + Send signature $\sigma=(A=\frac{C}{x+e}, e)$ back to user.

> [!NOTE]
>
> Current stataus
>
> + ***Public***:: $H_{nullifier}, X, \vec{\mathbf{m}}$
> + ***Verifier***: $<\mathcal{N}, 0xRelayerAddr>$  
> + ***Signer***: $x$, <$UserID$, $\mathcal{N}$, $\sigma$>
> + ***User***: $<0xUserAddr, nulifier$>,  $m_{\gamma}$, $\sigma$



## 4. User gen NIZK proof

+ User
  + Use **NIZK** proof to gen $\pi = (\bar{A}, \bar{B}, U, s, t)$ to prove that user do hold a valid signature $\sigma$ for $\vec{\mathbf{m}}|| m_{\gamma} || nullifier$ but hind the secret blind factor $m_\gamma$
  + Send $(\pi, nullifier, 0xUserAddr)$ to ***Validator***

> [!NOTE]
>
> + ***Public***: $H_{nullifier}, X, \vec{\mathbf{m}}$
>
> + ***Verifier***:: $<\mathcal{N}, 0xRelayerAddr, 0xUserAddr, nullifier, \pi>$, 
> + ***Signer***: $x$, <$UserID$, $\mathcal{N'}$, $\sigma$>
> + ***User***: $<0xUserAddr, nullifier$>, $\sigma$

## 5. Check the Signature and send the token

+ ***Validator*** 
  + Calc $\sum_i^{\ell - 2}m_iH_i$, this could be pre-compute because the constraints is already be hard-coded in the contract.
  + Calc $C_J = \sum_i^{\ell - 2}m_iH_i + m_{null}H_{null}$
  + Check  whether $(C_j, \pi)$ is valid.,  if not, revert.
  + Calc $\mathcal{N} = Hash(0xUserAddr,nullifier)$, require(KeyExist[$\mathcal{N}$]==true), and then set Deprecated=mapping(uint256=>bool)[N]=true
  + Call the Issuer's function, fn(`timestamp`, `0xUserAddr`) to issue the soulbind token to `0xUserAddr`

## 6. Summerize

- [x] The ***User*** can't use a same $\sigma$ at mutiple time, because of the nullifier
- [x] The ***Signer*** can't track where the User use the $\sigma$, because of the NIZK proof is unlinkable
