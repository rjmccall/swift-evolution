# Guidelines for Evolution Workgroups and Review Managers

This document lays out the detailed process used by Swift evolution workgroups (such as the [Language Workgroup](https://www.swift.org/community/#language-workgroup)) to take proposals through the [process of evolution review](https://github.com/apple/swift-evolution/blob/main/process.md).  It provides guidance for workgroups on how to decide which proposals should be reviewed, how to select a review manager to guide a review, and what options are available after a review and how to reach a decision between them.  It also provides guidance for review managers on their specific role and responsibilities during a review.

This is intended to be a companion document to a pair of documents directed at other audiences.  The first, tentatively titled "Evolution Overview and Guidelines for Community Members”, is intended for community members who are interested in participating in a pitch or evolution review authored by someone else.  Readers of this document will be expected to have read that document first; however, it has not yet been written.  The second, tentatively titled "Evolution Guidelines for Proposal Authors", is intended for community members interested in bringing their own proposal through the evolution process.  Readers of this document will not expected to have read that document, or vice-versa; it has also not yet been written.

Evolution reviews are associated with a particular workgroup that is responsible for guiding the evolution of that part of the Swift project. For the language and standard library, this is the [Language Workgroup](https://www.swift.org/community/#language-workgroup); other areas may be assigned to other workgroups.  The [Core Team](https://swift.org/community/#core-team) may act as the associated workgroup when an evolution review is desired but there isn't a more specific workgroup for that part of the project.  Hereafter in this document, the general term *workgroup* should be understood to be the associated workgroup for the specific proposal under consideration.

When a workgroup decides to consider a proposal for evolution review, it appoints a *review manager* to be its representative for that review. The review manager is typically a member of the workgroup. The review manager ensures the proposal meets the standards for review, handles review scheduling, writes public announcements about the review, oversees review discussion, compiles and summarizes review feedback for the workgroup’s deliberations, and conveys feedback from the workgroup to the authors. The rest of this document will describe all of these roles, as well as the process of selecting the review manager, in more detail.

## Initiating a review

A proposal author formally indicates that they would like the Swift project to initiate a review of their proposal by opening a (non-draft) pull request against the [swift-evolution repository](https://github.com/apple/swift-evolution/). Members of the associated workgroup will see the proposal and perform a *cursory review*. Since most proposals are associated with the Language Workgroup, the Language Workgroup is primarily responsible for monitoring this repository and informing other workgroups that there is a proposal for them to perform a cursory review of.

### Cursory review

The purpose of the cursory review is to ensure that the proposal meets the minimum standards to be considered for review:
- The proposal must be well-developed: the document should clearly explain what it is proposing and make a well-structured argument in favor of that proposal.
- The proposal must have been "thoroughly pitched": the community must have had an opportunity to provide feedback, and while discussion need not have completely ended, it should have reached some sort of steady state where it seems to largely be treading on familiar ground.
- The proposal must have an implementation: the basic requirements of the implementation should be known, its consequences (e.g. for source compatibility) should be understood, and people should be able to experiment with using it.

Workgroup members perform cursory review by ensuring that it has an appropriate label.  The following standard labels are available, but workgroup members may use other labels as they see fit:

- `Workgroup: Needs Development` means that the proposal document requires significant further development, either in its substance or in its presentation of the proposal. This label should be accompanied by detailed review comments (not necessarily from the workgroup member). Issues about the substance of the proposal should be raised in the pitch thread if they haven't yet been discussed there, and the PR review may simply refer to those posts.

- `Workgroup: Needs Pitch` means either that the proposal has not yet been pitched or that the pitch hasn't yet reached a steady state. If the proposal hasn't been pitched yet, the reviewer should call that out in a comment.

  This label should be periodically reconsidered while the proposal remains in pitch. If the reviewer decides that the pitch still hasn't reached a steady state, they should briefly describe the remaining controversy in a comment on the PR. For example: "Looks like there's still good discussion in the pitch about whether this feature should be able to throw an error."

- `Workgroup: Needs Implementation` means the proposal lacks an implementation or the implementation is lacking. This generally requires a comment explaining the problem.

- `Workgroup: Ready` means that the proposal is ready to be considered for review. If the workgroup agrees, it will be assigned a review manager, who will perform the *preliminary review*. See the "Assigning a review manager" section below.

If a workgroup member feels during cursory review that a proposal should simply be rejected without further review, they should bring that up with the full workgroup without assigning a label. See the "Rejecting a proposal without a review" section below.

Cursory review is meant to be a rapid and lightweight "triage" process which keeps proposal authors up to date about the current status and unmet expectations (if any) of their proposal. Cursory review should happen within two weeks of the proposal first being opened as a non-draft PR. Proposals that are currently being pitched should be re-reviewed at least every two weeks. Proposals that have been marked `Ready` should be assigned a review manager within two weeks; after that point, it becomes the review manager's responsibility to keep the authors aware of the situation (see below). Proposal authors who believe that their proposal has been overlooked, or who believe that it is ready for review despite the comments of cursory reviewers, may reach out to another member of the workgroup.

Proposal PRs are developed using the standard code-review processes of GitHub. Any member of the community may participate in this process. Workgroup members performing cursory review may need to ask other contributors who have worked in the area of the proposal to participate in the review or the pitch. Assigning the proposal PR to a reviewer does not imply anything about whether that reviewer will be the review manager for the proposal; in fact, it may imply that they are an expert in the proposal domain and should specifically *not* serve as the review manager so that they are more free to comment in the review.

### Rejecting a proposal without a review

Initiating a review is not an endorsement of the proposal by the workgroup. However, it is an endorsement of the idea that the proposal is at least worth the community’s time to consider. The workgroup does not have a responsibility to run every proposal for which a PR is opened, or even every proposal that meets the minimum standards.

* If the workgroup agrees that no proposal along the proposed lines will ever be accepted, that should be communicated to both the authors (either privately or in the proposal PR) and the community (in the pitch thread). The workgroup should explain its reasoning and the scope of its decision. The proposal PR may be closed with a link to the public discussion.

* Otherwise, if there is substantial agreement among workgroup members (not necessarily consensus, but enough to block acceptance) that they would not accept the proposal in its current form, a review should not be run. The members opposed to the proposal should engage with the pitch thread to explain their objections. This is considered to be part of the pitch phase, not a conclusive end to the proposal. The proposal PR should generally be labeled `Workgroup: Needs Development` rather than being immediately closed. If the authors respond in a way which satisfies the objections of the workgroup (either by modifying the proposal or convincing the workgroup members they are mistaken), then the proposal may be reviewed. If not, the proposal simply continues in the pitch phase, which is not limited in time. However, the workgroup may elect to close proposal PRs which appear to no longer be making progress towards review.

### Assigning a review manager

When a proposal reaches the `Workgroup: Ready` state, the workgroup performs a collective cursory review to decide whether it is ready to be considered for review. This may result in the proposal being returned for more development or being rejected without a review. If the workgroup decides that a proposal is indeed ready to be considered for review, it assigns the proposal a review manager.

Ideally, the review manager should be someone who hasn’t been been extensively engaged in the development of the proposal and is willing to remain neutral during the review. The review manager must be able to carry out their duties in a way that all sides will perceive as fair. If the review manager has expressed strong feelings about the proposal, community members may react badly to moderation decisions that seem to favor the review manager's side, or they may worry that the review manager will not fairly represent the views of opposing sides during workgroup deliberations. This can very quickly make the review a toxic environment.

That is not to say that the review manager may not have an opinion on the proposal's merits. Review managers are members of the workgroup and therefore likely to have opinions on most things of importance to the project, and reviews are certainly important to the project. But a review manager must be willing to treat all perspectives with an even hand during the review. Even if they disagree with a position, they must encourage that position's proponents to develop and present their arguments in the best way possible and then accurately summarize those arguments for the Language Workgroup. If a review manager decides at any point that they cannot continue to effectively serve, they should work with the rest of the workgroup to find a replacement.

Assigning a review manager does not commit the workgroup to actually running a review for the proposal. One of the review manager’s first responsibilities is to conduct the preliminary review advise the workgroup if the proposal is ready to review; see below.

The remainder of this document will address the reader (“you”) as if you were a review manager for a proposal.

### Pre-review procedure

After being appointed as a review manager, you should take the following the steps:

* Review the proposal document to ensure it meets project standards and will be productively reviewable by the community.
* Review the pitch thread(s) for the proposal to ensure that any major ideas brought up there are addressed by the proposal document.
* Work with the proposal authors to make any changes necessary in response to your reviews of the proposal and pitch, as well as to incorporate any early feedback received from the rest of the workgroup.
* Verify that there is an implementation of the proposal.
* Ensure that that the header fields of the proposal are correct in the pull request.
* Work with the workgroup and the proposal authors to schedule the review. The most important thing is that both you and the authors will be continually available during the review period, without more than a day's absence.
* When the review is about to begin, assign the proposal document an SE-NNNN number, update the pull request appropriately (see below), and then merge it into the Evolution repository.

### Preliminary review

Your goal in this preliminary stage is to set up a productive review that will deliver strong signal to the workgroup about how to proceed. You do this in three primary ways:

* Improve the substantive proposal.
* Improve the argumentation in the proposal document.
* Prepare the authors to engage productively with the review.

The first step in all of these things is for you yourself to review the entirety of the pitch thread and the proposal document.

When reviewing the pitch thread, you are looking for three main things:

* Questions and misunderstandings about what the proposal means, which may show weaknesses in the drafting of the proposal document. Even if something is ultimately explained in the document, if the explanation is out of order, readers may get confused and give up early.
* Objections and counter-proposals, which may identify be substantive weaknesses in the proposal, weaknesses in the proposal document’s arguments, or alternatives / trade-offs which should be more thoroughly discussed. Sometimes this comes down to a hard conflict between different ideas that the workgroup will have to resolve; the proposal document should be forthright about this and discuss why the proposal is the right way to go.
* Conversations about things not in the proposal, which may contain ideas that do need to be addressed. These ideas may either point to substantive weaknesses or future directions, depending on how they relate to the proposal, whether the proposal stands well without considering them, and/or whether accepting the proposal as-is will interfere with pursuing those ideas in the future. If you can't figure out the connection, consider just asking in the pitch thread; giving the commentator an opportunity to flesh out their argument can pay off in several different ways.

For each of these issues, consider the proposal document and decide whether there’s something worth bringing up with the authors. Use your judgment, keeping in mind that some comments really are just lazy or off-topic.

Sometimes the issues you identify require substantive changes to the proposal. Treat the pitch like a review and make sure that people in the pitch thread understand the changes and have time to react. Any feedback you can incorporate and build on during the pitch phase becomes background for the actual review, allowing it to run smoother and encouraging reviews to go deeper into the proposal.  A week of extra feedback can easily prevent a lot of confusion and maybe even a re-review.

Remember that the authors need to address confusion, objections, and counter-arguments not just in the pitch thread, but in the proposal document. People naturally dislike writing out the same thing twice, and so they can be tempted to hastily summarize for the proposal document, but this is backwards: the proposal document is the more authoritative place, and it deserves the best and clearest version of the explanation or argument. Future reviewers and programmers will not be referring to the pitch thread to understand the proposal. Even though the proposal itself is not substantively changing, you are simultaneously strengthening the argumentation in the proposal and preparing the authors better for the review period.

The proposal document is an artifact of the Swift project and, if accepted, speaks with the imprimatur of the project leadership. As such, it is expected to meet a high standard of clarity and professionalism.
* It must be well-written with a clear and easily-followed narrative that starts with a description of the problem and leads into its proposed solution.
* It should accurately describe and analyze the problem it addresses. If applicable, it should give a brief history of the problem in the project.  It should explain the impact of the problem on Swift developers.
* It must describe its proposed solution in adequate technical detail to be implemented.
* It should present a clear argument in favor of its proposed solution and, if necessary, against any alternatives that would conflict with it.
* It must be inoffensive and inclusive. This also covers any examples or sample code that might be included with the proposal. Authors sometimes use amusing or evocative examples to make their proposals feel less dry, and that's fine, but they must take care to not let these efforts perpetuate a stereotype or implicit bias. What seems playful to one person might be perceived by another (or in the future) as belittling or insulting. It is usually best for examples to be vague about the details of people (or anthropomorphized entities) if those details could be a source of offense.
* Sometimes it is necessary for proposals to criticize other projects. For example, a proposal to add a graphics library would need to discuss alternative library designs, and that would naturally include contrasting the proposal with a number of existing libraries and identifying reasons why it diverged from their designs in various ways. Such criticisms should be constructive, generous, and focused solely on technical issues relevant to the proposal.
It is your responsibility to review the proposal document to bring it in line with these standards before the review begins. Proposal authors face many pressures when bringing a proposal through the evolution process, and it is easy for them to overlook problems in the proposal document when they are focused on things like the implementation, the pitch thread, and the substance of the proposal. It is important to have a different set of eyes on the proposal. Feel free to ask for help from the workgroup if you’re not certain if something in the proposal is acceptable, or if you're trying to figure out how best to express difficult feedback to the author.

Once you feel you understand the proposal, it may also be useful to brief the workgroup about it. Some workgroup members may have participated in the pitch, but for others, this may be their first introduction to the details of this specific proposal. Pay attention to any questions and comments they have and treat them like you would treat feedback from the pitch thread.

### Interacting with the proposal authors

When talking to the proposal authors, you should be clear about the role you’re currently playing. Normally, when you are passing on the results of your review, you are merely making suggestions, asking for clarifications, and so on, as an ordinary member of the community. In principle, any member of the community could do the same review you have done and make the same comments; you simply have a responsibility to have done it. At the end of the day, the proposal belongs to the proposal authors, at least until it’s accepted. You have the authority as review manager to edit the header of the proposal and to make minor editorial changes to the body, but major edits or substantive changes to the document should be made by the proposal authors, and if they’re unwilling to make those changes, they don’t have to. If you feel that the proposal shouldn’t be reviewed until certain changes are made, or if you feel that the proposal authors are unlikely to engage productively during a review, you can argue that to the full workgroup. If the workgroup agrees that the proposal cannot currently be run, then you should communicate that decision back to the authors officially on behalf of the workgroup, and the authors can decide how they wish to proceed.

If your relationship to the proposal authors seems to be deteriorating, ask the workgroup for guidance. It may be better for someone else to take over as review manager. Alternatively, it may be necessary for the workgroup to take corrective action with the authors, such as reminding them of their responsibilities to engage with feedback as proposal authors and/or to follow the Code of Conduct as community members.

### Future Directions and Roadmaps

Most proposals should have at least some content in the Future Directions section. However, if the Future Directions section appears to be getting very large, or if the pitch thread is full of requests for greater scope in the proposal, or if the proposal feels like an early step towards something much larger, it may be appropriate to develop a roadmap for the larger feature area before proceeding with the review. The proposal can then clearly situate itself within that roadmap.

Generally, feature roadmap documents are solicited and approved by the workgroup outside of the evolution review process. This approval is not an endorsement of everything in the document, and it doesn’t set anything in stone, but it does signal general acceptance of the basic approach laid out by the roadmap.

If you believe that a feature roadmap may be appropriate as you help prepare a proposal for review, you should raise that with the workgroup. If the workgroup does decide to solicit a roadmap, the proposal review will need to be delayed until the roadmap is available.

### Bureaucratic details of managing the pre-review

* As a member of the workgroup, you should be able to add commits directly to the proposal PR, and that is generally what you will do.
* The filename of the proposal document should be `XXXX-text.md`, where the text is the current title of the proposal. Sometimes the proposal title changes from the first draft, and it's fine to update the filename during this phase. However, once the proposal has been assigned an SE number and merged into the swift-evolution repository, the filename should not be changed, even if the proposal is retitled.
* The `Proposal` field should contain a self-link to the current version of the proposal document using SE-NNNN as the link text.
* The `Authors` field should have the right pluralization and should link to each of the authors (their GitHub account, if they don’t have other preferences). The authors are responsible for deciding who counts as an author. However, you should not be listed as an author, even if you contributed extensive editorial help.
* The `Review Manager` field should link to you (generally, your GitHub account).
* The `Status` field should be `Awaiting Review` until the review begins.
* The `Roadmap` field should link to the discussion thread for the feature roadmap that this proposal is part of, if one exists.
* The `Implementation` field should link to the most important implementation PRs. It does not need to be comprehensive.
* The `Review` field should link to the pitch, like so: (pitch ([https://forums.swift.org/ ](https://forums.swift.org/))). Later links will be added the same way, separated by spaces, with concise but unambiguous link text.

## The review

The first round of review should generally last at least 10 days, including two full weekends. Reviews for larger proposals should be longer. Later rounds may have shorter review periods if the workgroup has greatly narrowed the scope of review.

You kick off the review by creating a review thread. Your post should follow the template used in previous reviews, updating the proposal name, link, dates, and their own contact information appropriately. You may also announce the review on other social media if you interact with Swift contributors there.

### Bureaucratic details of initiating the review

* Proposal document (round 1):
  * Make sure that pre-review details are right (above).
  * Figure out the next SE number. Remember to talk to your colleagues if more than one review is about to begin.
  * Edit the SE number into the filename of the document.
  * Edit the link target of the Proposal field to use the right SE number.
  * Edit the link text of the Proposal field to use the right SE number.
  * Change the Status field to **Active Review (Start Date...End Date, Year)**, in bold.
  * You will not be able to link to the review thread yet.
  * Preview the proposal document with all these changes.
  * If everything looks good, merge the PR.
* Create your review thread:
  * The thread title should be `SE-NNNN: Title of Proposal`. In later rounds of review, add `(Second Review)` (or whatever) immediately before the colon.
  * The category should be [Evolution > Proposal Reviews](https://forums.swift.org/c/evolution/proposal-reviews/21). You should have permission to create threads here. Do not create threads here for any other purpose.
  * Add any tags to the review that are meaningful.
  * Post content should generally follow the [review announcement template](https://github.com/apple/swift-evolution/blob/main/process.md#review-announcement) except as described below.
  * Feel free to tailor the salutation and closing as you see fit. Make sure it’s your name at the bottom.
  * Edit the proposal link to be correct. Just link to the main version of the proposal; don’t try to permalink the current revision, it’s more important that people browsing the forums for proposal information don’t accidentally read old revisions than that design historians see a consistent view of the proposal as it originally was at the start of the review.
  * Edit the dates of the proposal.
  * Edit the link to you (if applicable) in the part about contacting you directly, and make sure it reflects how you’d like people to contact you.
  * If there’s something special that the community needs to know procedurally about the review, add it in or after the first paragraph. In subsequent reviews, this will include summarizing the previous history of the proposal (including links to the previous reviews) and explaining how the proposal has been modified (if applicable) and what conclusions the workgroup has already reached (if applicable). If there is a limitation to the scope of the review — like if the workgroup has accepted certain parts, or if the review is just about a specific topic — be explicit about that. You can be honest about failures in the process, but you should discourage discussion of that in the review thread; you might need to make sure there’s somewhere else for that to go.
* Proposal document (round 2):
  * Edit the Review field to add a link to the review: ([review](https://forums.swift.org/)).
  * You’ll need to create a new PR to commit this.
  * Be sure to preview the document before you merge.
  * Merge if it looks good.
* Announce to the world that the review has started, if you have an appropriate personal channel that you wish to use for that.

### Responsibilities during the review

During the review, you are responsible for maintaining a collegial atmosphere in the review thread and for keeping the review on track. To do so, you will need to stay up to date on the thread and should not let it go unread for more than a day. Make sure you schedule the review during a time when this is possible for you.

Keeping the review on track isn’t always easy. Understanding the proposal under review often requires reviewers to explore its connections to other parts of the language and to try to understand it in the broader context of the evolution of the language. Such discussions are a legitimate part of the review and should not be discouraged. If an extended discussion seems to have gotten away from the original proposal, you should suggest that it should be moved into a separate thread. Contact a forums moderator if you need help moving posts between threads. Generally, though, try to use a loose hand about these things.

In contrast, you should be quick to discourage personal or inappropriate criticisms of the authors. Among other things, the motivations of the proposal authors are not a legitimate subject of review. If a discussion is turning into an argument, try to get involved early; having a third person in the conversation can lower the heat considerably. If it’s already gotten out of hand, though, don’t hesitate to flag posts for moderation; letting bad behavior sits unanswered quickly poisons the atmosphere, especially in review threads where people can feel a need to stay engaged.

You are not prohibited from participating in the review in your individual capacity, but you should always state clearly when you’re speaking for yourself and not as the review manager. You should also refrain from directly arguing for or against the proposal; if you’re finding that difficult, this is probably not the right review for you to be managing, and there is surely someone else on the workgroup willing to take over. People who disagree with you should not have to worry that you are not going to represent their views faithfully. There is a natural tendency for people to want to manage reviews in their area of expertise, but in fact the opposite is better: you should manage reviews that you’re less personally involved in. (They may even give you an opportunity to learn something interesting!)

Some reviewers may send you reviews by Swift Forums DM or by email. You should keep track of these reviews too—you will need them later. You should respond briefly just to acknowledge that you received the review.

## Concluding a review

When the review period has concluded, you should go back over all the reviews and prepare to discuss it with the workgroup. The workgroup should schedule a discussion about the review as soon as possible. The aim should be to reach a conclusion about a proposal within about a week of the end of its review, and that conclusion may require communication with the proposal authors, so the faster the process begins the better.

### Presenting review feedback

Your role in presenting the feedback is to represent it honestly and fairly, not to advocate for or against the proposal. (You can, of course, express your personal opinions separately from presenting the community feedback; just don’t let it flavor your interpretation of the feedback.) People who are uncomfortable with a proposal do not always provide cogent arguments against it, but that doesn’t mean their discomfort should be forgotten.

You do not need to exhaustively list all of the feedback. Generally, you should aim to do two things in your presentation of the feedback:

* Convey an accurate overall picture of the feedback. If there’s a lot of drive-by feedback disapproving of the proposal without making much of an argument for why, that’s still useful to know.
* Identify any interesting points that the workgroup ought to consider, regardless of how much support they received during the review. The most important feedback in the review might be something highly technical that most other reviewers won’t understand and so will overlook. This is one reason among many why the evolution process is not a vote.

Combining these is also valuable. Many reviewers may feel some way about the proposal, but maybe only a few of them managed to capture the reasons for those feelings in their review. It’s worthwhile to make a special effort to find feedback like that which may speak for many people. (Maybe there’s a way to address it!)

### Reaching a conclusion

After reviewing the feedback, the workgroup will discuss it and reach a conclusion about what to do.

There are six primary alternatives available to the workgroup at the end of a review:

* The proposal can be accepted without modification.
* The proposal can be accepted with modification and without further review. Generally, this should only be done in one of two following situations. Avoid the temptation to invent reasons not to submit the proposal for further review.
  * The community was adequately “briefed” about the possible modification and had an opportunity to respond. For example, the review might be a second or third review specifically to resolve one last thorny question, and the modification is to incorporate the decision the workgroup has made.
  * The modification only removes functionality from the proposal, and the workgroup feels that the proposal stands just fine without it.
* The proposal can be modified (or not) and immediately sent into a new review.
* The proposal can be modified (or not) and immediately have its review extended. This is more or less equivalent to running a new review, but it generally garners less attention, and so it is only appropriate for relatively minor modifications or questions.
* The proposal can be returned for revision. The proposal returns to the pitch phase; the authors may re-use the existing pitch thread or use a new one.
* The proposal can be outright rejected. This is appropriate when the workgroup has decided that no proposal along these lines will ever be accepted: the same condition as in the "Rejecting a proposal without a review" section, but after having run a review.

If the workgroup feels that it didn't get much useful feedback during the review, it's worth considering why. If the review thread fell prone to a lot of distractions, the presentation of the proposal may be flawed: it might be too big to meaningfully review in detail, or the community might be failing to understand it in context, or there might be an important issue that hasn't been appropriately addressed. Alternatively, the workgroup might be showing an unreasonable expectation of the review process: reviews are not generally an effective way to resolve open questions, and instead such proposals should be returned to the pitch phase, even if it's just to answer one nagging and trivial-seeming question, like what argument labels to give to an initializer. Sometimes the right decision is to not make a final decision yet and solicit more information. Evolution reviews should be scheduled with an understanding that they will take time and may require follow-on discussions and reviews. If there's only enough time in a release for a review if everything goes well, it probably shouldn't be expected to finish in the release unless it's a slam dunk.

Most proposals have room for improvement: to be more general, to be more expressive, to be easier to use, to be more safe. Most proposals also provide significant value just as they are. Understanding how the proposal relates to its future directions is a very important part of deciding whether it’s acceptable to ship as it stands. Easy-feeling implementation choices or extensions of features into new areas can very easily hamper the language from actually being able to pursue interesting future directions if they’re not done right away. Often, cutting or restricting a feature makes the language better in the long run by allowing it to be re-introduced when there’s time to do it without so many unintended consequences.

The workgroup is not bound to only consider the feedback received from the community; it may consider arbitrary things *de novo* in its deliberations. However, two considerations apply:

First, an evolution review provides value not just by directly evaluating the proposal, but also by evaluating the arguments for and against it. A deeper understanding of those arguments and their strengths and weaknesses gives reviewers a deeper understanding of the entire problem in ways that may lead back to a better proposal. Therefore, if the workgroup in its deliberations uncovers an important new idea which shapes how it thinks about the proposal, it has a responsibility to present that idea back to the review thread. The workgroup should do this by appointing someone to make a post in the thread presenting the idea; usually this will read something like, "The workgroup was talking about this, and we realized that...". The review should also be explicitly extended to give the community an opportunity to consider and respond to the new argument.

Second, workgroup members should resist the temptation to rely on being part of the review deliberations to avoid participating in the normal review. Workgroup members with interesting thoughts to share about a proposal should share them with the community and therefore give people the opportunity to consider and respond to those thoughts. This has many benefits. It allows those thoughts to better shape the community's thinking in general. It avoids unnecessary review extensions and extra work for the review manager under the policy above. It encourages members to consider how to improve the normal review experience. And finally, it discourages the idea that the workgroup is superior to the evolution process.

If the decision-making process is dragging out, remember to keep the proposal authors in the loop. It may also be useful to get their opinion directly about something that came up during the review; consider just inviting them to the workgroup meeting where their proposal will be discussed.

### Bureaucratic details of concluding the review

* When initiating a follow-up review of a proposal:
  * Start a new review thread for the proposal as described above. The introduction should discuss the workgroup’s conclusions, summarize any changes to the proposal, and describe any limitations which the workgroup would like to impose on the review.
  * Make a post in the last review/announcement thread about the proposal. This post should link to the new thread and say that the proposal is back in review.
  * Close the old thread.
  * Update the proposal document:
    * Change the status to `Active review` with the correct dates.
    * Add a link to the new review thread (as `(second review)` or whatever) to the `Review` field.
    * We don't generally rename the proposal file itself, even if the proposal has been renamed, because doing so will break old links.
* When extending the review of a proposal:
  * Make a post in the review thread discussing the workgroup’s conclusions, summarizing any changes, and describing any new limitations on the review.
  * Update the proposal document:
    * Change the status to `Active review` with the correct dates.
* For any other change:
  * Start a new announcement thread (in Evolution > Announcements):
    * The title should be `[New Status] SE-NNNN: Title of Proposal`
    * Describe the outcome of the review and thank the community for its contributions.
  * Close the review thread.
  * Update the proposal document:
    * Change the `Status` field to the new status
      * If the proposal is accepted, and the implementation has already been merged, you may mark it as `Implemented (Swift M.N)` instead. Otherwise, a separate PR should be made later for this.
    * Update the `Review` field with a link to the announcement thread (as `(acceptance)`, `(rejection)`, or `(returned for revision)`).

Generally, once you are assigned as the review manager of a proposal, you will remain the review manager over the course of all of its reviews. This is, of course, not a sentence of involuntary servitude; you’re welcome to ask if someone can take over from you. But if you do stay on as review manager, and a proposal goes back into pitch phase, your responsibilities basically recur: when a new swift-evolution PR is opened for the proposal, you need to review the new pitch comments, make sure the proposal is actually ready for review, and so on.
