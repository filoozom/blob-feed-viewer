<script lang="ts">
  import { Common, Chain, Hardfork } from "@ethereumjs/common";
  import { BlobEIP4844Transaction } from "@ethereumjs/tx";
  import { Address, initKZG, hexToBytes } from "@ethereumjs/util";
  import test from "/node_modules/kzg-wasm/dist/esm/wasm/kzg.wasm?url";
  import { onMount } from "svelte";
  import { createKZG } from "kzg-wasm";
  import { BrowserProvider, ZeroAddress } from "ethers";

  const chainId = 10200;

  const main = async () => {
    const provider = new BrowserProvider((window as any).ethereum, {
      chainId,
      name: "Chiado",
    });
    provider.send("wallet_switchEthereumChain", [{ chainId }]);

    console.log(test);

    const kzg = await createKZG();
    initKZG(kzg);

    const common = Common.custom(
      {
        name: "chiado",
        chainId,
        networkId: chainId,
      },
      {
        hardfork: Hardfork.Cancun,
        eips: [4844],
        customCrypto: { kzg },
      }
    );

    const txData = {
      data: "0x8d7cd6da",
      gasLimit: 50000,
      //to: Address.fromString("0xd4827E3f42A2ee43a1Af30f9772dBA8B53fc3074"),
      from: "0x3337f9995601c6E057f3C6dcf5d55f2eb5C744c0",
      to: ZeroAddress,
      chainId: 10200,
      blobsData: ["blablablalbla"],
    };

    const blobTx = BlobEIP4844Transaction.fromTxData(txData, { common });
    //const signer = await provider.getSigner();
    //signer.sendTransaction(blobTx.toJSON());

    console.log(blobTx.raw());
    (window as any).ethereum.request({
      method: "eth_sendRawTransaction",
      params: [blobTx.raw()],
    });
  };
</script>

<button on:click={main}>Post</button>
