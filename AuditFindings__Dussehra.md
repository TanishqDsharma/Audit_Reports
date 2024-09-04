# NOTES

# ChoosingRam Contract: ontract allows users to increase their values and select Ram, but only if they have not selected Ram before

->  If all chars are true users can select RAM
->  If the user has not selected Ram before 12 October 2024, then the Organizer can select Ram if not selected.
->  increaseValuesOfParticipants allows users to increase their values(or characteristics) and become Ram for the event. The values will never be updated again after 12 October 2024.
->  selectRamIfNotSelected - Allows the organizer to select Ram if not selected by the user.

# Dussehra contract:  allows users to enter the people who like Ram, kill Ravana, and withdraw their rewards.

-> enterPeopleWhoLikeRam allows users to enter the event like Ram by paying an entry fee and receiving the ramNFT.
-> killRavana—Allows users to kill Ravana, and the Organizer will receive half of the total amount collected in the event. This function will only work after 12 October 2024 and before 13 October 2024.
-> withdraw - Allows ram to withdraw their rewards.

# RamNFT: contract allows the Dussehra contract to mint Ram NFTs, update the characteristics of the NFTs, and get the characteristics of the NFTs.

This contract allows the Dussehra contract to mint Ram NFTs, update the characteristics of the NFTs, and get the characteristics of the NFTs.

setChoosingRamContract - Allows the organizer to set the choosingRam contract.
mintRamNFT - Allows the Dussehra contract to mint Ram NFTs.
updateCharacteristics - Allows the ChoosingRam contract to update the characteristics of the NFTs.
getCharacteristics - Allows the user to get the characteristics of the NFTs.
getNextTokenId - Allows the users to get the next token id.
NFTs are minted with the following characteristics:

ram: address of user
isJitaKrodhah: false // JitaKrodhah means one who has conquered anger
isDhyutimaan: false // Dhyutimaan means one who is intelligent
isVidvaan: false // Vidvaan means one who is knowledgeable
isAatmavan: false // Aatmavan means one who is self-controlled
isSatyavaakyah: false // Satyavaakyah means one who speaks the truth
