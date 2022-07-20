# Evolution Guidelines for Review Managers

This post is a companion to “Evolution Overview and Guidelines for Community Members” [note: not yet fully drafted], and it assumes you have read that document. It explains the process followed by the Language Workgroup for evolution reviews, and it explains the role and responsibilities of the review manager.

A review manager is the representative of the Language Workgroup for a specific review and is typically a member of the workgroup. They are chosen by the workgroup when it decides that a proposal is ready to be considered for review. They convey feedback from the workgroup to the authors, ensure the proposal meets the standards for review, handle review scheduling, write public announcements about the review, oversee review discussion, and compile and summarize review feedback for the workgroup’s deliberations.

Review managers may have an opinion on the proposal’s merits, but they are expected to carry out their duties in a way that all sides will perceive as fair. Even if they disagree with a position, they must try to prompt that position's proponents to develop and present their arguments as well as they can and then accurately summarize those arguments for the Language Workgroup. If a review manager decides at any point that they cannot continue to serve as review manager, they should work with the rest of the workgroup to find a replacement.

## Initiating a review

A proposal author formally indicates that they would like the Language Workgroup to initiate a review of their proposal by opening a (non-draft) pull request against the [swift-evolution repository](https://github.com/apple/swift-evolution/).  The workgroup then has a responsibility to consider the proposal for review.  Workgroup members periodically monitor the active pull requests and mark proposals that are ready to be considered for review with the “Language Workgroup: Ready for Consideration” label.  (GitHub does not currently allow labels to be protected, but other community members should not use this label.)  The workgroup will then schedule a preliminary discussion of the proposal.  This preliminary discussion should ideally begin within a few weeks of the PR being opened.

Initiating a review is not an endorsement of the proposal by the Language Workgroup.  However, it is an endorsement of the idea that the proposal is at least worth the community’s time to consider.  The workgroup does not have a responsibility to run every proposal for which a PR is opened.

* If there is a sentiment among workgroup members that they are unlikely to accept a proposal in its current form, it should not be run.  The members opposed to the proposal should engage with the authors (either in the pitch thread or on the proposal pull request) and explain their objections.  If the authors respond in some way that satisfies the objections of the workgroup, the proposal may still be considered for review.  If they do not, that should not be taken as conclusive: this engagement is simply a continuation of the pitch phase of the proposal, which is not limited in time.  The workgroup may close the swift-evolution PR if no progress appears to be being made for an extended period, however.

* If the workgroup wishes to make a stronger stand against a proposal, it may decide that no proposal along the proposed lines will ever be accepted.  This should be politely but firmly communicated to the authors and the community, and the PR may be closed immediately.

If the Language Workgroup decides that a proposal is ready for review, it will assign the proposal a review manager. Ideally, the review manager should be someone who hasn’t been extensively engaged in the pitch and is willing to stay neutral during the review, but sometimes that isn’t possible.  After all, review managers are usually workgroup members, and workgroup members are chosen in part for their expertise; it would be strange to expect a workgroup member to be completely dispassionate about something important to the project, as many reviews are.  Nonetheless, the workgroup should aim to choose a review manager who will be perceived as even-handed during the review.

Assigning a review manager does not commit the workgroup to actually running a review for the proposal.  Part of the review manager’s role is to advise the workgroup if the proposal is ready to review; see below.

The remainder of this document will address the reader (“you”) as if you were a review manager for a proposal.

### Pre-review procedure

After being appointed as a review manager, you should take the following the steps:

* Review the proposal document to ensure it meets project standards and will be productively reviewable by the community.
* Review the pitch thread(s) for the proposal to ensure that any major ideas brought up there are addressed by the proposal document.
* Work with the proposal authors to make any changes necessary in response to your reviews of the proposal and pitch, as well as to incorporate any early feedback received from the rest of the Language Workgroup.
* Verify that there is an implementation of the proposal.
* Ensure that that the header fields of the proposal are correct in the pull request.
* Work with the Language Workgroup and the proposal authors to schedule the review.  The most important thing is that both you and the authors will be continually available during the review period, without more than a day's absence.
* When the review is about to begin, assign the proposal document an SE-NNNN number, update the pull request appropriately (see below), and then merge it into the Evolution repository.

### Pre-review review

Your goal in this preliminary stage is to set up a productive review that will deliver strong signal to the Language Workgroup about how to proceed.  You do this in three primary ways:

* Improve the substantive proposal.
* Improve the argumentation in the proposal document.
* Prepare the authors to engage productively with the review.

The first step in all of these things is for you yourself to review the entirety of the pitch thread and the proposal document.

When reviewing the pitch thread, you are looking for three main things:

* Questions about what the proposal means, which may be weaknesses in the drafting of the proposal document.  (Or the questioner may not have read the proposal.)
* Objections and counter-proposals, which may be substantive weaknesses in the proposal, weaknesses in the proposal document’s arguments, or alternatives / trade-offs which should be more thoroughly discussed.  (Or the objector may simply be expressing a fundamental disagreement that ultimately the workgroup will have to resolve.)
* Conversations about things not in the proposal, which may contain ideas that should be addressed in the proposal.  These ideas may either be substantive weaknesses or future directions, depending on how they relate to the proposal, whether the proposal stands well without considering them, and/or whether accepting the proposal as-is will interfere with pursuing those ideas in the future.  (Or the contributors may misunderstand what the proposal’s about, or they may simply wish to talk about something other than the topic at hand.)

For each of these, read the proposal document and decide whether there’s something worth bringing up with the authors. Do this especially if the authors did not engage with those posts in the pitch thread. The authors may have reasonable arguments against many of the ideas brought up; that’s great, but make sure those arguments are captured in the proposal document.  Even if the proposal itself does not change, you have both strengthened the argumentation in the proposal and better prepared the authors for the review period.

The proposal document is an artifact of the Swift project and, if accepted, speaks with the imprimatur of the project leadership.  As such, it is expected to meet a high standard of clarity and professionalism.  The document should be well-written.  It should accurately describe the problem it addresses.  It should describe its proposed solution in adequate technical detail to be implemented.  It should present a clear argument in favor of its proposed solution and, if necessary, against any alternatives that would conflict with it.  Examples, metaphors, and sample code in the proposal should be inclusive and free of offensive or suggestive material.  If it is necessary to criticize an existing project in the proposal, that criticism should be constructive, generous, and focused solely on technical issues.  It is your responsibility to review the proposal to bring it in line with these standards before the review begins.  Feel free to ask for help from the workgroup if you’re not certain how best to express something.

Once you feel you understand the proposal, it may also be useful to brief the Language Workgroup about it.  Some workgroup members may have participated in the pitch, but for others, this may be their first introduction to the details of this specific proposal.  Pay attention to any questions and comments they have and treat them like you would treat feedback from the pitch thread.

### Interacting with the proposal authors

When talking to the proposal authors, you should be clear about the role you’re currently playing.  Normally, when you are passing on the results of your review, you are merely making suggestions, asking for clarifications, and so on, as an ordinary member of the community.  In principle, any member of the community could do the same review you have done and make the same comments; you simply have a responsibility to have done it.  At the end of the day, the proposal belongs to the proposal authors, at least until it’s accepted.  You have the authority as review manager to edit the header of the proposal and to make minor editorial changes to the body, but major edits or substantive changes to the document should be made by the proposal authors, and if they’re unwilling to make those changes, they don’t have to.  If you feel that the proposal shouldn’t be reviewed until certain changes are made, or if you feel that the proposal authors are unlikely to engage productively during a review, you can argue that to the full Language Workgroup.  If the Workgroup agrees that the proposal cannot currently be run, then you should communicate that decision back to the authors officially on behalf of the Workgroup, and the authors can decide how they wish to proceed.

If your relationship to the proposal authors seems to be deteriorating, ask the Language Workgroup for guidance.  It may be better for someone else to take over as review manager.  Alternatively, it may be necessary for the Workgroup to take corrective action with the authors, such as reminding them of their responsibilities to engage with feedback as proposal authors and/or to follow the Code of Conduct as community members.

### Future Directions and Roadmaps

Most proposals should have at least some content in the Future Directions section.  However, if the Future Directions section appears to be getting very large, or if the pitch thread is full of requests for greater scope in the proposal, or if the proposal feels like an early step towards something much larger, it may be appropriate to develop a roadmap for the larger feature area before proceeding with the review.  The proposal can then clearly situate itself within that roadmap.

Generally, roadmap documents are solicited and approved by the Language Workgroup outside of the evolution review process.  This approval is not an endorsement of everything in the document, and it doesn’t set anything in stone, but it does signal general acceptance of the basic approach laid out by the roadmap.  Talk to the Workgroup about how to proceed if you think a roadmap is appropriate for a feature.

### Bureaucratic details of managing the pre-review

* As a member of the Language Workgroup, you should be able to add commits directly to the proposal PR, and that is generally what you will do.
* The filename of the proposal document should be `XXXX-text.md`, where the text is the current title of the proposal. Sometimes the proposal title changes from the first draft.
* The `Proposal` field should contain a self-link to the current version of the proposal document using SE-NNNN as the link text.
* The `Authors` field should have the right pluralization and should link to each of the authors (their GitHub account, if they don’t have other preferences).  The authors are responsible for deciding who counts as an author.  However, you should not be listed as an author, even if you contributed extensive editorial help.
* The `Review Manager` field should link to you (generally, your GitHub account).
* The `Status` field should be Awaiting Review until the review begins.
* The `Implementation` field should link to the most important implementation PRs.  It does not need to be comprehensive.
* The `Review` field should link to the pitch, like so: (pitch (https://forums.swift.org/)).  Later links will be added the same way, separated by spaces, with concise but unambiguous link text.

## The review

The first round of review should generally last at least 10 days, including two full weekends.  Reviews for larger proposals should be longer.  Later rounds may have shorter review periods if the Language Workgroup has greatly narrowed the scope of review.

You kick off the review by creating a review thread. Your post should follow the template used in previous reviews, updating the proposal name, link, dates, and their own contact information appropriately. You may also announce the review on other social media if you interact with Swift contributors there.

### Bureaucratic details of initiating the review

* Proposal document (round 1):
    * Make sure that pre-review details are right (above).
    * Figure out the next SE number.  Remember to talk to your colleagues if more than one review is about to begin.
    * Edit the SE number into the filename of the document.
    * Edit the link target of the Proposal field to use the right SE number.
    * Edit the link text of the Proposal field to use the right SE number.
    * Change the Status field to **Active Review (Start Date...End Date, Year)**, in bold.
    * You will not be able to link to the review thread yet.
    * Preview the proposal document with all these changes.
    * If everything looks good, merge the PR.
* Create your review thread:
    * The thread title should be `SE-NNNN: Title of Proposal`.  In later rounds of review, add `(Second Review)` (or whatever) immediately before the colon.
    * The category should be [Evolution > Proposal Reviews](https://forums.swift.org/c/evolution/proposal-reviews/21).  You should have permission to create threads here.  Do not create threads here for any other purpose.
    * Add any tags to the review that are meaningful.
    * Post content should generally follow the template (https://forums.swift.org/c/evolution/proposal-reviews/21) except as described below
    * Feel free to tailor the salutation and closing as you see fit.  Make sure it’s your name at the bottom.
    * We generally just link to the proposal from “SE-NNNN” and drop the “the proposal is available here” bit.  Yes, we should edit the template, nobody’s gotten around to it.
    * Edit the proposal link to be correct.  Just link to the main version of the proposal; don’t try to permalink the current revision, it’s more important that people browsing the forums for proposal information don’t accidentally read old revisions than that design historians see a consistent view of the proposal as it originally was at the start of the review.
    * Edit the dates of the proposal.
    * Edit the link to you (if applicable) in the part about contacting you directly, and make sure it reflects how you’d like people to contact you.
    * If there’s something special that the community needs to know procedurally about the review, add it in or after the first paragraph.  In subsequent reviews, this will include summarizing the previous history of the proposal (including links to the previous reviews) and explaining how the proposal has been modified (if applicable) and what conclusions the Language Workgroup has already reached (if applicable).  If there is a limitation to the scope of the review — like if the workgroup has accepted certain parts, or if the review is just about a specific topic — be explicit about that.  You can be honest about failures in the process, but you should discourage discussion of that in the review thread; you might need to make sure there’s somewhere else for that to go.
* Proposal document (round 2):
    * Edit the Review field to add a link to the review: (review (https://forums.swift.org/)).
    * You’ll need to create a new PR to commit this.
    * Be sure to preview the document before you merge.
    * Merge if it looks good.
* Announce to the world that the review has started, if you have an appropriate personal channel that you wish to use for that.

### Responsibilities during the review

During the review, you are responsible for maintaining a collegial atmosphere in the review thread and for keeping the review on track.  To do so, you will need to stay up to date on the thread and should not let it go unread for more than a day. Make sure you schedule the review during a time when this is possible for you.

Keeping the review on track isn’t always easy.  Understanding the proposal under review often requires reviewers to explore its connections to other parts of the language and to try to understand it in the broader context of the evolution of the language.  Such discussions are a legitimate part of the review and should not be discouraged.  If an extended discussion seems to have gotten away from the original proposal, you should suggest that it should be moved into a separate thread.  Contact a forums moderator if you need help moving posts between threads.  Generally, though, try to use a loose hand about these things.

In contrast, you should be quick to discourage personal or inappropriate criticisms of the authors.  Among other things, the motivations of the proposal authors are not a legitimate subject of review.  If a discussion is turning into an argument, try to get involved early; having a third person in the conversation can lower the heat considerably.  If it’s already gotten out of hand, though, don’t hesitate to flag posts for moderation; letting bad behavior sits unanswered quickly poisons the atmosphere, especially in review threads where people can feel a need to stay engaged.

You are not prohibited from participating in the review in your individual capacity, but you should always state clearly when you’re speaking for yourself and not as the review manager.  You should also refrain from directly arguing for or against the proposal; if you’re finding that difficult, this is probably not the right review for you to be managing, and there is surely someone else on the workgroup willing to take over.  People who disagree with you should not have to worry that you are not going to represent their views faithfully.  There is a natural tendency for people to want to manage reviews in their area of expertise, but in fact the opposite is better: you should manage reviews that you’re less personally involved in.  (They may even give you an opportunity to learn something interesting!)

Some reviewers may send you reviews by Swift Forums DM or by email. You should keep track of these reviews too—you will need them later.  You should respond briefly just to acknowledge that you received the review.

## Concluding a review

When the review period has concluded, you should go back over all the reviews and prepare to discuss it with the Language Workgroup.  The workgroup should schedule a discussion about the review as soon as possible.  The aim should be to reach a conclusion about a proposal within about a week of the end of its review, and that conclusion may require communication with the proposal authors, so the faster the process begins the better.

### Presenting review feedback

Your role in presenting the feedback is to represent it honestly and fairly, not to advocate for or against the proposal.  (You can, of course, express your personal opinions separately from presenting the community feedback; just don’t let it flavor your interpretation of the feedback.)  People who are uncomfortable with a proposal do not always provide cogent arguments against it, but that doesn’t mean their discomfort should be forgotten.

You do not need to exhaustively list all of the feedback.  Generally, you should aim to do two things in your presentation of the feedback:

* Convey an accurate overall picture of the feedback.  If there’s a lot of drive-by feedback disapproving of the proposal without making much of an argument for why, that’s still useful to know.
* Identify any interesting points that the workgroup ought to consider, regardless of how much support they received during the review.  The most important feedback in the review might be something highly technical that most other reviewers won’t understand and so will overlook.  This is one reason among many why the evolution process is not a vote.

Combining these is also valuable.  Many reviewers may feel some way about the proposal, but maybe only a few of them managed to capture the reasons for those feelings in their review.  It’s worthwhile to make a special effort to find feedback like that which may speak for many people.  (Maybe there’s a way to address it!)

### Reaching a conclusion

After reviewing the feedback, the Language Workgroup will discuss it and reach a conclusion about what to do.

There are six primary alternatives available to the workgroup at the end of a review:

* The proposal can be accepted without modification.
* The proposal can be accepted with modification and without further review.  Generally, this should only be done in one of two following situations.  Avoid the temptation to invent reasons not to submit the proposal for further review.
    * The community was adequately “briefed” about the possible modification and had an opportunity to respond. For example, the review might be a second or third review specifically to resolve one last thorny question, and the modification is to incorporate the decision the workgroup has made.
    * The modification only removes functionality from the proposal, and the workgroup feels that the proposal stands just fine without it.
* The proposal can be modified (or not) and immediately sent into a new review.
* The proposal can be modified (or not) and immediately have its review extended.  This is more or less equivalent to running a new review, but it generally garners less attention, and so it is only appropriate for relatively minor modifications.
* The proposal can be returned for revision.
* The proposal can be outright rejected.

The workgroup is not bound to only consider the feedback received from the community and may consider arbitrary things de novo in its discussion.  With that said, it’s always better for workgroup members with interesting ideas about a proposal to instead raise them during pitch or review.  Resist the temptation to wait until the workgroup meeting to share your thoughts.

Most proposals have room for improvement: to be more general, to be more expressive, to be easier to use, to be more safe.  Most proposals also provide significant value just as they are.  Understanding how the proposal relates to its future directions is a very important part of deciding whether it’s acceptable to ship as it stands.  Easy-feeling implementation choices or extensions of features into new areas can very easily hamper the language from actually being able to pursue interesting future directions if they’re not done right away.  Often, cutting or restricting a feature makes the language better in the long run by allowing it to be re-introduced when there’s time to do it without so many unintended consequences.

Try to keep proposal authors in the loop, especially if the decision-making process is taking more than a week.

### Bureaucratic details of concluding the review

* When initiating a follow-up review of a proposal:
    * Start a new review thread for the proposal as described above.  The introduction should discuss the workgroup’s conclusions, summarize any changes to the proposal, and describe any limitations which the workgroup would like to impose on the review.
    * Make a post in the last review/announcement thread about the proposal.  This post should link to the new thread and say that the proposal is back in review.
    * Close the old thread.
    * Update the proposal document:
        * Change the status to `Active review` with the correct dates.
        * Add a link to the new review thread (as (second review) or whatever) to the `Review` field.
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
            * If the proposal is accepted, and the implementation has already been merged, you may mark it as `Implemented (Swift M.N)` instead.  Otherwise, a separate PR should be made later for this.
        * Update the `Review` field with a link to the announcement thread (as `(acceptance)`, `(rejection)`, or `(returned for revision)`).

Generally, once you are assigned as the review manager of a proposal, you will remain the review manager over the course of all of its reviews.  This is, of course, not a sentence of involuntary servitude; you’re welcome to ask if someone can take over from you.  But if you do stay on as review manager, and a proposal goes back into pitch phase, your responsibilities basically recur: when a new swift-evolution PR is opened for the proposal, you need to review the new pitch comments, make sure the proposal is actually ready for review, and so on.
