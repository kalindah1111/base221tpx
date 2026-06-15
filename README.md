# base221tpx
0x81b057dD6B2bAA9584aE5f129bAd27415A18D1E7
import time
from collections import defaultdict, deque
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 20
QUIET_ACTIVITY_MAX = 10
SPIKE_ACTIVITY_MIN = 50
SCORE_THRESHOLD = 3

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Detecting pre-activity accumulators...\n")

    last_block = w3.eth.block_number

    token_history = defaultdict(
        lambda: deque(maxlen=5)
    )

    token_wallets = defaultdict(set)

    wallet_scores = defaultdict(int)

    while True:

        try:

            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                logs = w3.eth.get_logs({
                    "fromBlock": current_block - WINDOW_BLOCKS,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })

                current_activity = defaultdict(int)
                current_wallets = defaultdict(set)


                for log in logs:

                    token = log["address"]

                    sender = decode_address(
                        log["topics"][1]
                    )

                    receiver = decode_address(
                        log["topics"][2]
                    )


                    current_activity[token] += 1


                    if sender != ZERO:
                        current_wallets[token].add(sender)

                    if receiver != ZERO:
                        current_wallets[token].add(receiver)


                for token, activity in current_activity.items():

                    history = token_history[token]

                    if len(history) >= 3:

                        previous = sum(history) / len(history)

                        if (
                            previous <= QUIET_ACTIVITY_MAX
                            and activity >= SPIKE_ACTIVITY_MIN
                        ):

                            for wallet in token_wallets[token]:
                                wallet_scores[wallet] += 1


                    token_wallets[token].update(
                        current_wallets[token]
                    )

                    history.append(activity)


                print(
                    f"\nBlocks "
                    f"{current_block - WINDOW_BLOCKS}"
                    f" -> {current_block}"
                )


                for wallet, score in wallet_scores.items():

                    if score >= SCORE_THRESHOLD:

                        print("📌 Pre-Activity Accumulator")
                        print("Wallet:", wallet)
                        print("Score:", score)
                        print()


                last_block = current_block


            time.sleep(3)


        except Exception as e:

            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
