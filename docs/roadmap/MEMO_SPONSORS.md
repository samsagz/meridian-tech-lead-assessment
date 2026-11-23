# Memo: Home Essentials and the GenAI assistant

**To:** Regional Director of Home Insurance (Home Essentials) · Chief Commercial Officer (GenAI assistant)
**From:** Technical Lead, Direct Digital Channel · November 2026

## Home Essentials: what you get and when

- Customers can quote and buy in **Mexico by 26 April, Colombia by 26 May and Chile by 25 June 2027**; advisors start one week earlier. We go live a few days before your dates because the last two days of each month are the busiest, and we make no changes then.
- A quote in five questions or less, and payment in the same session.
- Most policies arrive in under three minutes; every policy arrives within 24 hours, or the customer gets a refund.
- Quotes keep working when Salesforce is down (about 3% of each month), because prices are calculated in advance every morning.

## Trade-offs we made

- We build the product once for the three countries: 45–55 person-weeks instead of 75–90. The current Mexican products move to the new platform one at a time, in the second half of 2027.
- Before we build, we close the causes of the 30 September incident.
- The first quarter uses all our capacity, so our vendor, Nimbus Digital, works on the product from February. If Mexico is not ready on 15 March, we will propose that it launches with Colombia at the end of May.

## GenAI assistant: our answer

**No for customers in the first quarter of 2027. Yes to a test, with a decision on 15 July 2027.** In March we test the assistant with real questions from the Mexican contact center; in May and June, agents use it internally. With a go, Mexican customers can ask about coverages in the last quarter of 2027. Pre-filling the quote comes later, with a new type of AI model that suits that task better.

Why not now: the approval covers only internal use, recommending a coverage can be regulated advice, and we do not know how many calls it would avoid. Coverage calls cost about USD 886,000 per year; the assistant should cost under USD 15,000 per year to run, never over USD 100,000, and we turn it off if it does not pay for itself after three months. Meanwhile, simple coverage summaries help customers now and improve the assistant later.

## What we need from you

| What | Who | By |
|---|---|---|
| Support our request to the Salesforce platform team (prices calculated in advance, one integration developer for six weeks), and agree on the Mexico fallback to 31 May | Regional Director | 15 Dec 2026 |
| Product decisions: answer ranges for the five questions, and how fresh prices must be | Regional Director, with Product | 15 Jan 2027 |
| Legal approval to serve the three countries from one platform | Regional Director, with Legal | 31 Jan 2027 |
| Support for the new terms in the Nimbus Digital contract | Both | 31 Jan 2027 |
| Start the Security and Compliance review for customer use | Chief Commercial Officer | 31 Jan 2027 |
| 300 real coverage questions and a group of agents for the pilot | Chief Commercial Officer | 1 Mar 2027 |
