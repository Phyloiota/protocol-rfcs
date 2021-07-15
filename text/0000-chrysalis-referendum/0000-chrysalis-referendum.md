+ Feature name: chrysalis-referendum
+ Start date: 2021-07-11
+ RFC PR: To be created...

# Summary

This RFC defines ontangle voting for Chrysalis Phase 2. Specifically, it describes the structure and flow of a referendum, the format of voting data to be included in the transaction, and all endpoints on the node side.

# Motivation

With IF offering the community to decide the fortune of the unclaimed tokens <link to blogpost>, IOTA needs to provide a system for community members to create and vote on referenda. In community calls, we figured out that the only way to archive a fair referendum without having to invest years in R&D would be a simple token-based system. With this, any token holders can use their tokens to vote for one of the options, one token, one vote. Based on an online poll, the majority of the community wants to continue having community referenda. Therefore the system also needs to support future referenda, as well as multiple at the same time.

The described system tries to address the following points:

* No protocol fork required, as voting data is stored in the data (indexation) payload of a value transaction
* Protection against flash loan attacks, where an attacker buys a lot of tokens to game referenda and dump them instantly when it's over.
  * This is archived by adding a third factor to voting weight, which is holding time
* No token locking required
* Optional multiple-choice vote (choose n out of m options)
* Tokens stay in your wallet at all time


# Detailed design

## Referendum flow

A referendum has 4 different stats

### Upcomming

Any referendum will start here. The question and all answer options are set and final. Also, the start of the referendum, the start of the staking phase and the end of the referendum are set. Each referendum has a unique 256-bit hash as a unique identifier, which can be used to share the vote.

Referenda are neither automatically added to all nodes, nor are they broadcasted on the network. Instead, any node might give its users (or the owner only) the option to manually add referenda to track. Therefore the creation of a referendum should set the voting start at least a few days into the future and make sure there are enough nodes that tally up the votes. Referenda can only be added to nodes during this phase.

### Commencing

Once the starting milestone defined in the referendum has been reached, the referendum enters the commencing phase. During this phase, votes can already be cast and are recorded by the node, which constantly updates the number of tokens voting for every option whenever a milestone comes in. However, they have no weight yet.

Whatever time is set as the beginning, there will always be a place where it will be at a very inconvenient time (e.g. the middle of the night). This phase grants fairness to every timezone, as it does not matter when in this phase you submit your vote. Also, it can be used to draw public attention and spread the word. Ideally, this phase should last for a few days or even a week so everyone has a chance to give his vote 100% of its weight.

Any votes for a referendum that is still upcoming or unknown to the node are ignored. This does not cause validation to fail. For example, if someone votes for 3 referenda at once but only one is known to the node, it will count the vote normally for one and ignore the other two.

### Staking

After the commencing phase is over, the staking phase will begin. Every time a milestone comes in, the node will first update the number of tokens voting for each option (if necessary) and then add this amount to the total count for each option. The addition will first happen one milestone after this phase begins, as we always credit for the duration between two milestones. The last addition happens at the end milestone, attributing weight for the last period.

This adds a time factor to the vote. If you buy tokens and vote while the referendum is ongoing, you only get weight for the remaining time. Similiar, if you decide to sell, you will miss out on the weight for the remaining time. If you decide to change your opinion, your old opinion will get the weight of the elapsed time since the staking phase started, while your new opinion will get the weight of the remaining time.

Example: Assume Alice has 2i and does not vote at all. Now she sends their tokens to Bob 10000 milestones after the beginning of the staking period (approx. 1d 4h). Bob instantly votes for option A. After another 20000 milestones have passed, he sends the tokens to Charlie, who instantly votes option B. 5000 Milestones later, the referendum ends. Assuming the total supply consists of 2 iotas the outcome would be:
*     2i*10000=20000 are not counted in the voting
*     2i*20000=40000 for Build (80%)
*     2i* 5000=10000 for Burn (20%)
*     Turnout: 50000/70000=71,4% 

In reality, those numbers would be a lot higher, enough to exceed the 64-bit integer limit given enough participation.

### Ended

Once the end milestone has been received and the total counters have been updated a final time, the referendum is over. New votes are ignored, just as if the node did not know the referendum at all. 

## Referendum format

Users can submit votes to any nodes that allow it. A referendum consists out of the question and multiple options to choose from, as well as 3 milestones, where the referendum starts, when staking starts and when it is over. There is also a number on how many options may be chosen, which allows multiple-choice votes if >1. Last but not least there is a description field that might contain additional data

### Detailed format

<table>
    <tr>
        <th>Name</th>
        <th>Type</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>Question Length</td>
        <td>uint8</td>
        <td>
        The length of the question string in bytes
        </td>
    </tr>
      <tr>
        <td>Question</td>
        <td>String</td>
        <td>
        The referendum question in UTF-8 format
        </td>
    </tr>
      <tr>
        <td>Begin Milestone</td>
        <td>uint32</td>
        <td>
        The milestone where the referendum begins
        </td>
    </tr>
    </tr>
      </tr>
      <tr>
        <td>Stake Milestone</td>
        <td>uint32</td>
        <td>
        The milestone where the referendum enters staking
        </td>
    </tr>
    </tr>
      </tr>
      <tr>
        <td>End Milestone</td>
        <td>uint32</td>
        <td>
        The milestone where the referendum ends
        </td>
    </tr>
    <tr>
        <td>Option Count</td>
        <td>uint8</td>
        <td>The count of options proceeding</td>
    </tr>
    <tr>
        <td valign="top">Voting option <code>oneOf</code></td>
        <td colspan="2">
            <details open="true">
                <summary>Voting option</summary>
                <blockquote>
                A votable option for the referendum
                </blockquote>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Option ID</td>
                        <td>uint8</td>
                        <td>
                        The ID of the option. Counted from 0 to 19.
                        </td>
                    </tr>
                    <tr>
                        <td>Option length</td>
                        <td>uint8</td>
                        <td>
                        The length of the option text in bytes
                        </td>
                    </tr>
                    <tr>
                        <td>Option Text</td>
                        <td>String</td>
                        <td>
                        The text of the option in UTF-8 format
                        </td>
                    </tr>
                  </table>
          </details>
  </td>
    <tr>
        <td>Votable Options</td>
        <td>uint8</td>
        <td>The amount of options that can be chosen at once (multiple-choice)</br>This is a maximum, the user might still choose less options</td>
    </tr>
    <tr>
        <td>Additional Information Length</td>
        <td>uint16</td>
        <td>
        The length of the question string in bytes. Set to 0 if empty
        </td>
    </tr>
      <tr>
        <td>Additional Information</td>
        <td>String</td>
        <td>
        Additional information in UTF-8 format. Field does not exist if the length was set to 0
        </td>
    </tr>
</table>

<i>The BLAKE2b-256 hash of this object is the voting ID, which is used as the UUID of the referendum</i>

### Syntactical validation

* All fields are required
* All fields must contain a value greater than zero (numeric) or not be empty (text); except for the additional information field
* Question length must not exceed 250 bytes
* Option count must be 1 ≤ x ≤ 20 options
  * Option length must not exceed 100 bytes
  * Options must be ordered ascending by their IDs
* All options together must not exceed 1000 bytes
* The referendum must start in 1 ≤ x ≤ 600000 milestones (~10 weeks)
* The staking period has to start 360 ≤ x ≤ 600000 milestones after the referendum started
* The referendum has to end 360 ≤ x ≤ 600000 milestones after staking started
* The number of options to be chosen at once must be ≤ the total option count
* The additional information field must not exceed 1000 bytes and is the only field that might be empty

Those limits ensure that nobody can waste a large amount of disk space by putting gibberish. Also, questions and answer options are limited, so there shouldn't be too much trouble rendering it.

## Voting Format

Voting data will be sent via the indexation payload of a value transaction. To vote, you send a transaction to yourself, specifying your decisions.

### Criteria for a valid voting transaction

* Must be a value transaction
* Inputs must all come from the same address. Multiple inputs are allowed
* Has a singular output going to the same address as all input addresses.
  * Output Type 0 (SigLockedSingleOutput) and Type 1 (SigLockedDustAllowanceOutput) are both valid for this
* The Index must be "IOTAVOTE"
* The data must be parseable

### Structure of the voting data

<table>
    <tr>
        <th>Name</th>
        <th>Type</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>Vote count</td>
        <td>uint16</td>
        <td>
        The amount of votes following. Must be at least one
        </td>
    </tr>
    <tr>
        <td valign="top">Votes<code>oneOf</code></td>
        <td colspan="2">
            <details open="true">
                <summary>Voting option</summary>
                <blockquote>
                A votable option for the referendum
                </blockquote>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>ReferendumID</td>
                        <td>Array&lt;byte&gt;[32]</td>
                        <td>
                        The BLAKE2b-256 hash of the referendum to cast votes for
                        </td>
                    </tr>
                    <tr>
                        <td>Vote count</td>
                        <td>uint8</td>
                        <td>
                        The amount of votes to be cast for this referendum. Must be between 1-20.
                        </td>
                    </tr>
                    <tr>
                        <td>Vote options</td>
                        <td>Array&lt;uint8&gt;[n]</td>
                        <td>
                        The list of optionIDs, the length being equal to the previous value
                        </td>
                    </tr>
                  </table>
          </details>
      </td>
</table>

### Syntactic validation

If parsing of the data fails, the transaction is ignored for voting. The limit for the number of votes within a singular transaction is 65536, but the message size limit will be hit way beforehand.

Validation of a singular vote passes, if

* The ID has been added to the node for voting
* The referendum is in progress (not in "Upcoming" or "Ended" state)
* At least one optionID was chosen, but not more than allowed
* No optionID was chosen multiple times
* All optionIDs are existing for the specified referendum.

If a singular vote fails, others are unaffected. For example, if one vote is for a referendum that the node is unaware of, others will still behave normally.

## Node endpoints

To be added

# Drawbacks

To vote, you always have to send tokens to yourself, so every time you receive tokens, you have to send them to yourself again.
It is also technically possible to have different votes for different UTXOs on a single address.

# Rationale and alternatives

To be added

# Unresolved questions

To be added
