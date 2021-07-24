+ Feature name: chrysalis-referendum
+ Start date: 2021-07-11
+ RFC PR: To be created...

# Summary

This RFC defines ontangle voting for Chrysalis Phase 2. Specifically, it describes the structure and flow of a referendum, the format of voting data to be included in the transaction, and all endpoints on the node side.

# Motivation

Due to the IOTA Foundation offering the community to decide the fortune of the unclaimed tokens <link to blogpost>, IOTA needs to provide a system for community members to create and vote on referenda. In community calls, we figured out that the only way to archive a fair referendum without having to invest years in R&D would be a simple token-based system. With this, any token holders can use their tokens to vote for one of the options on each question, one token, one vote. During the weekly meetings, the majority of the community agreed to have further ongoing community referenda to establish a system and decide on the features of this system through voting. Therefore the system also needs to support future referenda, as well as multiple at the same time.

The described system tries to address the following points:

* No protocol fork is required as voting data is stored in the data (indexation) payload of a value transaction
* Protection against flash loan attacks, where an attacker buys a lot of tokens to game referenda and dumps them instantly after the referenda
  * This is archived by adding a third factor to voting weight, which is holding time
* No token locking required
* Referendums can have multiple questions (allows compact voting without much overhead)
* Tokens remain in your wallet at all time


# Detailed design

## Referendum flow

A referendum has 4 different stats

### Upcomming

Any referendum will start here. The creator specifies the questions and all answer options, as well as beginning, start of the holding phase and end of the referendum (see next section). A ID is created by hashing the entire referendum data, therefore the same combination always results in the same ID.

Referenda are neither automatically added to all nodes, nor are they broadcasted on the network. Instead, the referendum must be added to multiple nodes manually. Because submitting the same referendum to multiple nodes always results in the same ID, it is possible to compare data of multiple nodes. This also provides security, as users can verify they are actually voting for the correct referendum (e.g. because the ID is published on Github).

As a result, the creator of a referendum should set the voting start at least a few days into the future and make sure there are enough nodes that track it. Referenda can only be added to nodes during this phase.

### Commencing

Once the starting milestone defined in the referendum has been reached, the referendum enters the commencing phase. During this phase, votes can already be cast and are recorded by the node, which constantly updates the number of tokens voting for every option on each question whenever a milestone comes in. However, they have no weight yet.

Whatever time is set as the beginning, there will always be a place where it will be at a very inconvenient time (e.g. the middle of the night). This phase grants fairness to every timezone, as it does not matter when in this phase you submit your vote. Also, it can be used to draw public attention and spread the word. Ideally, this phase should last for a few days or even a week so everyone has a chance to give his vote 100% of its weight.

Any votes for a referendum that is still upcoming or unknown to the node are ignored. This does not cause validation to fail. For example, if someone votes for 3 referenda at once but only one is known to the node, it will count the vote normally for one and ignore the other two.

### Holding

After the commencing phase is over, the holding phase will begin. Every time a milestone comes in, the node will first update the number of tokens voting for each option on every question (if there were changes in the ledger) and then add this amount to the total count for each option. The addition always adds weight for the following period, since token balances cannot change until a new milestone is received. It will happen first on the Holding Start Milestone and last one milestone before the vote ends.

This adds a time factor to the vote. If you buy tokens and vote while the referendum is ongoing, you only get weight for the remaining time. Similiar, if you decide to sell, you will miss out on the weight for the remaining time. If you decide to change your opinion, your old opinion will get the weight of the elapsed time since the holding phase started, while your new opinion will get the weight of the remaining time.

Example: Assume Alice has 2i and does not vote at all. Now she sends their tokens to Bob 10000 milestones after the beginning of the holding period (approx. 1d 4h). Bob instantly votes for option A. After another 20000 milestones have passed, he sends the tokens to Charlie, who instantly votes option B. 5000 Milestones later, the referendum ends. Assuming the total supply consists of 2 iotas the outcome would be:
*     2i*10000=20000 are not counted in the voting
*     2i*20000=40000 for Build (80%)
*     2i* 5000=10000 for Burn (20%)
*     Turnout: 50000/70000=71,4% 

In reality, those numbers would be a lot higher, enough to exceed the 64-bit integer limit given enough participation.

### Ended

Once the end milestone has been received, the referendum is over. New votes are ignored, just as if the node did not know the referendum at all. 

## Referendum format

Users can submit referenda to any nodes that have the endpoint open. A referendum consists out of one or multiple questions, which each have multiple options to choose from. It also has 3 milestone numbers stored, when the referendum starts, when holding starts and when it is over. The referendum, the question and each answer has a information field, which may contain additional information about it.

### Detailed format

<table>
    <tr>
        <th>Name</th>
        <th>Type</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>Referendum Name Length</td>
        <td>uint8</td>
        <td>
        The length of the referendum name in bytes
        </td>
    </tr>
      <tr>
        <td>Referendum Name</td>
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
        <td>Holding Start Milestone</td>
        <td>uint32</td>
        <td>
        The milestone where the referendum enters holding phase
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
        <td>Question Count</td>
        <td>uint8</td>
        <td>The count of questions proceeding</td>
    </tr>
    <tr>
        <td valign="top">Voting question <code>oneOf</code></td>
        <td colspan="2">
            <details open="true">
                <summary>Voting question</summary>
                <blockquote>
                A question that can be voted on
                </blockquote>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Question Idx</td>
                        <td>uint8</td>
                        <td>
                        The index of the question. Counted from 1 to 255.
                        </td>
                    </tr>
                    <tr>
                        <td>Question Text Lengh</td>
                        <td>uint8</td>
                        <td>
                        The length of the question in bytes
                        </td>
                    </tr>
                    <tr>
                        <td>Question name</td>
                        <td>String</td>
                        <td>
                        The text of the question in UTF-8 format
                        </td>
                    </tr>
                    <tr>
                        <td>Option Count</td>
                        <td>uint8</td>
                        <td>
                        The amount of options following
                        </td>
                    </tr>
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
                                The ID of the option. Counted from 1 to 255.
																													 		</td>
																															</tr>
																															<tr>
																																<td>Option Text Length</td>
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
																															<tr>
																																<td>Option Information Length</td>
																																<td>uint16</td>
																																<td>
																																The length of the option information text in bytes
																																</td>
																															</tr>
																															<tr>
																																<td>Option Information</td>
																																<td>String</td>
																																<td>
																																Additional information about the option UTF-8 format
																																</td>
																															</tr>
																														</table>
																												</details>
																										</td>
                        <tr>
                        <td>Question Information Length</td>
                        <td>uint16</td>
                        <td>
                        The length of the question information text in bytes
                        </td>
                    </tr>
                    <tr>
                        <td>Question Information</td>
                        <td>String</td>
                        <td>
                        Additional information about the question in UTF-8 format
                        </td>
                    </tr>
                  </table>
          </details>
  </td>
    <tr>
        <td>Referendum Information Length</td>
        <td>uint16</td>
        <td>
        The length of additional information in bytes. Set to 0 if empty
        </td>
    </tr>
      <tr>
        <td>Referendum Information</td>
        <td>String</td>
        <td>
        Additional information in UTF-8 format. Field does not exist if the length was set to 0
        </td>
    </tr>
</table>

<i>The BLAKE2b-256 hash of this object is the referendum ID, which is used as the UUID of the referendum</i>

### Syntactical validation

* The entire object must not exceed 1 million bytes
* The referendum must start in 1 ≤ x ≤ 600000 milestones (~10 weeks)
* The holding period has to start 360 ≤ x ≤ 600000 milestones after the referendum started
* The referendum has to end 360 ≤ x ≤ 600000 milestones after holding started
* All additional information fields must not exceed 1000 bytes
* Questions must be ordered in ascending order by their index
* Answer options must be ordered in ascending order by their ID

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
        <td>Referendum count</td>
        <td>uint8</td>
        <td>
        The amount of referenda following. Must be at least one
        </td>
    </tr>
    <tr>
        <td valign="top">Referendum vote<code>oneOf</code></td>
        <td colspan="2">
            <details open="true">
                <summary>Referendum vote</summary>
                <blockquote>
                A vote for a referendum
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
                        How many votes are being cast. Must be equal to the amount of questions in the referendum
                        </td>
                    </tr>
                    <tr>
                        <td>Vote options</td>
                        <td>Array&lt;uint8&gt;[n]</td>
                        <td>
                        A list of optionIDs, one for each question.
                        First item is for question 1, second one for question 2, ...
                        </td>
                    </tr>
                  </table>
          </details>
      </td>
</table>

### Syntactic validation

If parsing of the data fails, the transaction is ignored for voting.

Validation of a singular referendum vote passes, if

* The ID has been added to the node for voting
* The referendum is in progress (not in "Upcoming" or "Ended" state)
* The amount of votes casted is equal to the amount of questions in the referendum

If a singular referendum vote fails, others are unaffected. For example, if one vote is for a referendum that the node is unaware of, others will still behave normally.

If there is no such optionID for a question, the question is ignored and no vote will be cast for this question only. As 0 does never exist as an optionID, it can be used to skip a question, but still vote for all others.

## Node endpoints

<i>This is more of a quick overview. A more detailed version is in work</i>

* GET /referendum/ : Lists all referenda, returning their UUID, the referendum name and status. Status `(upcomming,commencing,holding,ended)` must be used as a filter


* GET /referendum/{referendumID} : Gives a quick overview over the referendum. This does not include the questions or current standings.
* GET /referendum/{referendumID}/questions : Returns the entire vote with all questions, but not current standings.
* GET /referendum/{referendumID}/questions/{questionIdx} : Returns information and vote options for a specific question
* GET /referendum/{referendumID}/status : Returns the amount of tokens voting and the weight on each option of every question
* GET /referendum/{referendumID}/status/{questionIdx} : Return the amount of tokens voting for each option on the specified question

* POST /referendum/ : Submit a vote to track. By default not reachable from outside
* DELETE /referendum/{referendumID}: Removes a tracked vote.

# Drawbacks

* To vote, you always have to send tokens to yourself, so every time you receive tokens, you have to send them to yourself again. It is also technically possible to have different votes for different UTXOs on a single address.
* The standard referendum vote length would be 8 (Index) +2 (Header) +32 (ID) +1 (option count) +1 (option) = 44 additional bytes, which is rather long and might therefore increase the POW requirement for the transaction. However, additional questions on a referendum only take one extra byte.
* If someone has multiple addresses, they could be linked together if he cast a vote for all addresses at the same time. Firefly should probably have some sort of delay to prevent this.

# Rationale and alternatives

* Another proposal was to create a network fork, and send all tokens either to address A for Build or B for Burn. However, this is more of a one-time solution as it cannot be reused for future referenda.
* Tokens could also be locked for a few days to prevent flash-loan attacks. However, this would turn away possible voters.
* The referendum could check for how long the tokens have not been moved and attribute voting power based on this time. However, this would turn away voters that constantly use IOTA, even if the tokens mostly stay in someones wallet.

# Unresolved questions

* Are the limits chosen reasonable?
* Do we need additional endpoints?
