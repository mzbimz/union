<script lang="ts">
import Input from "$lib/components/ui/Input.svelte"
import Label from "$lib/components/ui/Label.svelte"
import Skeleton from "$lib/components/ui/Skeleton.svelte"
import { balancesStore } from "$lib/stores/balances.svelte"
import { wallets } from "$lib/stores/wallets.svelte.ts"
import { transferData } from "$lib/transfer/shared/data/transfer-data.svelte.ts"
import { Option, pipe } from "effect"
import { formatUnits, toHex } from "viem"

type Props = {
  type: "source" | "destination"
  disabled?: boolean
}
let { type, disabled = false }: Props = $props()

let chainWallet = $derived.by(() => {
  if (Option.isSome(transferData.sourceChain)) {
    return wallets.getAddressForChain(transferData.sourceChain.value)
  }
  return Option.none()
})

function allDataReadyForBalance() {
  return (
    Option.isSome(transferData.sourceChain)
    && Option.isSome(transferData.baseToken)
    && Option.isSome(transferData.baseTokenBalance)
    && Option.isSome(transferData.baseTokenBalance.value.balance)
  )
}

// Fetch denom only for selected balance
$effect(() => {
  const rec = Option.all({
    chain: transferData.sourceChain,
    address: transferData.derivedSender,
    denom: Option.map(transferData.baseToken, x => x.denom),
  })
  if (Option.isSome(rec)) {
    balancesStore.fetchBalances(rec.value.chain, rec.value.address, rec.value.denom, "2 minutes")
  }
})

let displayBalance = $derived.by(() => {
  if (
    Option.isSome(transferData.sourceChain)
    && Option.isSome(transferData.baseToken)
    && Option.isSome(transferData.baseTokenBalance)
  ) {
    const bal = transferData.baseTokenBalance.value.balance
    if (Option.isSome(bal)) {
      const baseToken = transferData.baseToken.value
      const decimals = baseToken.representations[0]?.decimals ?? 0
      const raw = bal.value
      return formatUnits(BigInt(raw), decimals)
    }
  }
  return "" // Not loaded yet
})

function setMaxAmount() {
  if (Option.isNone(transferData.baseToken)) {
    return
  }
  if (Option.isNone(transferData.baseTokenBalance)) {
    return
  }
  if (Option.isNone(transferData.baseTokenBalance.value.balance)) {
    return
  }
  if (Option.isNone(transferData.sourceChain)) {
    return
  }

  const baseToken = transferData.baseToken.value
  const rawBalance = BigInt(transferData.baseTokenBalance.value.balance.value)
  const decimals = baseToken.representations[0]?.decimals ?? 0
  const isUbbn = transferData.sourceChain.value.universal_chain_id === "babylon.bbn-1"
    && baseToken.denom === toHex("ubbn")

  const BABY_SUB_AMOUNT = 20n * 10n ** BigInt(decimals)

  const maxUsable = isUbbn && rawBalance > BABY_SUB_AMOUNT
    ? rawBalance - BABY_SUB_AMOUNT
    : rawBalance

  const formattedAmount = formatUnits(maxUsable, decimals)

  transferData.raw.updateField("amount", formattedAmount)

  const inputElement = document.getElementById("amount") as HTMLInputElement
  if (inputElement) {
    inputElement.value = formattedAmount
    inputElement.dispatchEvent(new Event("input", { bubbles: true }))
  }
}
</script>

<div class="w-full">
  {#if type === "source"}
    <div class="flex w-full justify-between items-center text-xs gap-1 pb-1">
      <div class="flex gap-1">
        <Label>BALANCE:</Label>
        {#if !transferData.raw.source || !transferData.raw.asset}
          0
        {:else if !allDataReadyForBalance()}
          <Skeleton class="h-3 w-16 inline-block" />
        {:else}
          {displayBalance}
        {/if}
      </div>
      <button
        class="cursor-pointer text-xs text-zinc-700 hover:opacity-80"
        onclick={setMaxAmount}
      >
        MAX
      </button>
    </div>
  {/if}
  <!-- XXX: label should take a snippet -->
  <Input
    id="amount"
    type="text"
    required
    disabled={!transferData.raw.asset || disabled}
    label=""
    autocorrect="off"
    placeholder="Enter amount"
    spellcheck="false"
    autocomplete="off"
    inputmode="decimal"
    data-field="amount"
    onbeforeinput={(event) => {
      const { inputType, data, currentTarget } = event
      const { value } = currentTarget
      const proposed = value + (data ?? "")

      const maxDecimals = pipe(
        transferData.baseToken,
        Option.flatMap((x) => Option.fromNullable(x.representations[0]?.decimals)),
        Option.getOrElse(() => 0),
      )

      const validShape = /^\d*[.,]?\d*$/.test(proposed)
      const validDecimalsDot = !proposed.includes(".")
        || proposed.split(".")[1].length <= maxDecimals
      const validDecimalsComma = !proposed.includes(",")
        || proposed.split(",")[1].length <= maxDecimals
      const isDelete = inputType.startsWith("delete")
      const validDecimals = validDecimalsComma && validDecimalsDot
      const noDuplicateLeadingZeroes = !proposed.startsWith("00")

      const allow = isDelete || (validDecimals && validShape && noDuplicateLeadingZeroes)

      if (!allow) {
        event.preventDefault()
      }
    }}
    autocapitalize="none"
    pattern="^[0-9]*[.,]?[0-9]*$"
    value={transferData.raw.amount}
    class="h-14 text-center text-lg"
    oninput={(event) => {
      transferData.raw.updateField("amount", event)
    }}
  />
</div>
