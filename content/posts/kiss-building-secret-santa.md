---
title: "Keep It Simple, Seriously: Building a no-frills Secret Santa assigner"
date: 2022-11-25T10:32:48Z
draft: false
---

Author's note: Yes, I haven't posted in a year. I published my [MSc
thesis](https://hdl.handle.net/10216/144775), and have started work with Amazon
Web Services. But this post is quick and fun. And I'll be memeing Amazon's
[Leadership Principles](https://www.amazon.jobs/en/principles)™️ along the way,
as a homage.

---

Okay, let's get started and set the scene: every year, my family gets together
for Chrismas eve, and a tradition that we have, since we are so many, is to do a
Secret Santa - everyone, kids excluded, draws another member of the extended
family to gift to; then, instead of having to gift everyone a bunch of useless
trinkets, they can give a better thought out present! Seems simple enough ☺️.

Having started working, and so being an "adult" for the first time this year, I
was excited to take part in this tradition, but I was dismayed to learn that it
was being threatened by a death by a thousand cuts. With so many people now
participating, it was getting unwieldy to draw names for everyone, since it
needed to be done, by hand, on an ocasion where at least one person of every
household of the family was present. Plus:

- The people that usually do the drawing, usually my beautiful mother or one my
  gracious aunts, often needed to be the 'gamemaster' and know some of the
  assignments to control for restrictions in the assignment.
- Having many participants means that some of them ended not being present on
  the actual Christmas eve for the unwrapping, cheapening the experience a bit.
  And that the thing itself took too much time, which made the kids cranky.

Okay, so let's work backwards™️. The main papercuts are:

- Not everyone that usually participated would be present.
- It's hard to make a draw in person.
- Too much knowledge on one person about the assignments.

The first point was the harder to solve, on human terms. We are making the
difficult decision of only allowing the ones present on Dec 24 to participate.
This will allow the experience to be more focused on the ones present there.

Now to the second and third ones. We need automation to solve this. And, to
totally, definitelly honor the invent and simplify™️ motto, we're not gonna use
one of the billion existing apps. So I started prototyping.

Now, a thing you should know about me is that I sometimes obsess over little
things and get stuck in the analysis paralysis stage of design. So, for building
my secret santa assigner, I started thinking of several layers to automate more
than one draft, how to approach this (basically NP-complete) problem in the most
efficient way, and so on, and so forth.

And so I was not making any real progress.

Until it hit me: Keep It Simple, Seriously!

I made an experiment - I got my trusty Deno environment up, and quickly modelled
the problem. You have a bunch of names, and a set of restrictions (in our case,
you never are the secret santa to anyone in your household). So the input was
something like this, at the bare minimum level of detail:

```ts
type SecretSantaInput = {
  people: Set<string>;
  pools: Set<Set<string>>;
};
```

I put some representative input in a JSON file, imported it, and made the
simplest algorithm I could think of - just making random candidate assingments
until one fit the restrictions. To my surprise, it always worked in a fraction
of a second, in about 10 iterations.

Which means that in a couple more hours, I added several more features:

- Use [Amazon SES](https://aws.amazon.com/ses/) to send emails to everyone with
  their assignment. Frugality™️ cause it means I can use my personal account's
  Free Tier allowance and I don't need to figure out how to send SMTP emails via
  my Gmail account.
- Pool together messages to the same email account, cause older folks will not
  know how to check their email.
- Validate input with [zod](https://github.com/colinhacks/zod) to make sure I
  don't screw up the input.

And it's done! No frills, exceedingly customer-obsessed™️, and makes my computer
nerd happy knowing that I made it myself.

Now I just need to start a business from (in some way that beats all the free
versions available and doesn't conflict with my non-compete clauses 🙃). The
first business decision I will make is to give away
[the code](https://github.com/joaonmatos/secret-santa) for free.

Happy holidays, everyone!
