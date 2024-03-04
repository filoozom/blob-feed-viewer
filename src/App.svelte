<script lang="ts">
  import { getDefaultProvider, isAddress, AbiCoder } from "ethers";
  import { Gapless5 } from "@regosen/gapless-5";
  import { onMount } from "svelte";

  import "@regosen/gapless-5/gapless5.css";

  const API = "https://rpc-gbc.chiadochain.net";
  const genesis = 1665396300;
  const provider = getDefaultProvider("https://rpc.chiadochain.net");

  let player: Gapless5;
  let currentAddress: string | undefined;
  let stopped = false;

  const sleep = (ms: number) =>
    new Promise((resolve) => setTimeout(resolve, ms));

  const hexToUint8Array = (hex: string) => {
    const cleaned = hex.startsWith("0x") ? hex.substring(2) : hex;
    const decoded = cleaned
      .match(/../g)
      ?.flatMap((h: string, i: number) => parseInt(h, 16));

    return Uint8Array.from(decoded ?? []);
  };

  const toVersionedHash = async (commitment: string) => {
    // Hex to buffer
    const buffer = hexToUint8Array(commitment);

    // Hash
    const hashBuffer = await crypto.subtle.digest("SHA-256", buffer);

    // Buffer to hex
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const hashHex = hashArray
      .map((bytes) => bytes.toString(16).padStart(2, "0"))
      .join("");

    // Return KZG version + hash without first byte
    return "0x01" + hashHex.substring(2);
  };

  const handleBlob = (hex: string, address: string) => {
    // Detect ogg header
    if (!hex.startsWith("0x004f676753")) {
      return;
    }

    // Convert to Uint8Array
    const cleaned = hex.startsWith("0x") ? hex.substring(2) : hex;
    const decoded = cleaned
      .match(/../g)
      ?.flatMap((h: string, i: number) =>
        i % 32 === 0 ? [] : [parseInt(h, 16)]
      );
    const buffer = Uint8Array.from(decoded ?? []);

    // Create Blob
    const blob = new Blob([buffer], { type: "audio/ogg; codecs=opus" });
    const url = URL.createObjectURL(blob);

    // Short-circuit
    if (address !== currentAddress) {
      return;
    }

    // Push to play queue
    player.addTrack(url);

    // Play last track
    if (!stopped && !player.isPlaying()) {
      console.log("Playing last track");
      player.onload = (path, loaded) => {
        if (path === url && loaded) {
          player.play();
        }
      };
      player.gotoTrack(url);
    }
  };

  const fetchBlob = async (
    slot: number,
    versionedHash: string,
    address: string
  ) => {
    for (let i = 0; i < 3; i++) {
      try {
        const response = await fetch(
          API + `/eth/v1/beacon/blob_sidecars/${slot}`
        );
        const data = await response.json();

        if (address !== currentAddress) {
          return;
        }

        for (const blob of data.data) {
          const hash = await toVersionedHash(blob.kzg_commitment);
          if (hash === versionedHash) {
            handleBlob(data.data[0].blob, address);
            return;
          }
        }

        console.warn(`Blob ${versionedHash} not found in slot ${slot}`);
        return;
      } catch (err) {
        await sleep(250);
      }
    }
  };

  const fetchLogs = async (address?: string) => {
    player?.removeAllTracks();

    if (!isAddress(address)) {
      return;
    }

    const logs = await provider.getLogs({
      address,
      topics: [
        "0x920bafc9b698db43ceb2a24a000b11adbc6618546595516600022ec103554902",
      ],
      fromBlock: 0,
    });

    for (const log of logs) {
      if (address !== currentAddress) {
        return;
      }

      const tx = await provider.getTransaction(log.transactionHash);
      const timestamp = Number(new AbiCoder().decode(["uint256"], log.data)[0]);
      const slot = (timestamp - genesis) / 5;

      if (!tx || tx.type !== 3) {
        continue;
      }

      for (const hash of tx.blobVersionedHashes ?? []) {
        await fetchBlob(slot, hash, address);
      }
    }
  };

  onMount(() => {
    player = new Gapless5({
      guiId: "player",
      useWebAudio: true,
      useHTML5Audio: false,
    });

    player.onstop = () => (stopped = true);
    player.onpause = () => (stopped = true);
    player.onplay = () => (stopped = false);
  });

  $: fetchLogs(currentAddress);
</script>

<input type="text" bind:value={currentAddress} />

<div>
  <div id="player" />
</div>
