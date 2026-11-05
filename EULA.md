# SQL Blame End User License Agreement

Version 1.0, effective from the first release of SQL Blame. This agreement is between you (the person or the
organization that uses SQL Blame) and Robert Weikert,
running a sole proprietorship in Poland (the "Author", contact: rw.devtools@gmail.com). The Author is the licensor of
SQL Blame. JetBrains, which sells SQL Blame licenses through the JetBrains Marketplace, is not a party to this
agreement. By installing or using SQL Blame you accept it.

## 1. The software

"SQL Blame" means the IntelliJ IDEA plugin, the Java agent it adds to your applications, the Explain Worker and their
documentation, in every version, including updates.

## 2. License

1. You buy a license for SQL Blame through the JetBrains Marketplace. The license type (personal or commercial), the
   number of users, the subscription period (monthly or annual) and any perpetual fallback license are those stated
   in the confirmation e-mail JetBrains sends you for the purchase (the "Confirmation"). The purchase itself,
   payment, renewal and refunds are governed by the JetBrains Marketplace terms between you and JetBrains.
2. With a valid license or during the free trial, the Author grants you a limited, non-exclusive, non-transferable,
   worldwide right to install and use SQL Blame within the limits of the Confirmation and its documentation, for any
   lawful purpose, including commercial work, and to keep backup copies for archival purposes. SQL Blame is
   licensed, not sold. The number of people using it at the same time may not exceed the number of users you
   bought.
3. Beta builds and the trial are provided free of charge for evaluation, "as is" like every version, and a later
   version may need a license. The Author has no obligation to provide support, updates or bug fixes, for any
   version.
4. Without the Author's written permission you may not sell, rent, sublicense or redistribute SQL Blame, remove or
   work around its license check, or modify, decompile, disassemble or reverse-engineer it or create derivative works
   of it, except to the extent the law allows this regardless of this agreement.
5. SQL Blame and all rights in it stay with the Author. Everything not expressly granted here is reserved.
6. The Author may change the features, the price and these terms for future versions and may end development or
   support. A license you already paid for keeps the rights stated in its Confirmation.

## 3. Your applications and your data

1. SQL Blame adds a Java agent to applications you run, debug, test or attach it to, and records the SQL they
   execute, including parameter values that may contain personal data. You decide where it runs. It is meant for
   development and test environments. You are responsible for having the right to run it on an application, to
   attach it to a JVM, to connect to its debug port and databases and to process the data it captures, including
   under data protection law, and for any use on production systems or on systems you are not allowed to inspect.
2. The captured data stays on your computer as described in the [privacy statement](PRIVACY.md). The Author receives
   none of it and does not process it on your behalf, except error reports and messages you choose to send.
3. SQL Blame sends EXPLAIN statements to your databases. Explain Analyze also runs the query (a `SELECT`, in a
   read-only transaction that is rolled back, after you confirm it) and costs what the query costs. Use these
   features only on databases where that is acceptable.

## 4. No warranty

SQL Blame is provided "as is" and "as available", without implied warranties. The Author does not warrant that it is
free of errors or viruses, that it runs without interruption, that errors will be corrected, that it captures every
query or that its numbers and findings are complete or correct. Check its findings before you act on them. Updates
are provided at the Author's discretion.

## 5. Limitation of liability

To the extent the law allows, the Author is not liable for any direct or indirect costs, losses or damages arising
from the use of SQL Blame, including lost profits, lost or damaged data, business interruption and consequential
damage, and the Author's total liability is limited to the fees you paid for your SQL Blame license through the
JetBrains Marketplace in the three months before the claim. JetBrains is not liable to you under this agreement in any
way, as it is
not a party to it. Nothing in this agreement limits liability that cannot be limited by law, such as liability for
intentional damage, or the mandatory rights of consumers.

## 6. Term

This agreement applies as long as you use SQL Blame. Your right to use it ends when your license or trial ends, or
when you break this agreement. Then stop using SQL Blame and remove it.

## 7. Changes

The Author may update this agreement for new versions of SQL Blame. The version that applies is the one published
with the version you install.

## 8. Law

Polish law applies, without its conflict-of-law rules and without the UN Convention on Contracts for the
International Sale of Goods. Disputes go to the courts competent for the Author's registered place of business. If you
are a consumer, you keep the protection of the mandatory law of the country where you live and may also use the
courts there.

## 9. Final provisions

1. You will not use SQL Blame in breach of applicable law, including export control and sanctions law.
2. If a provision of this agreement is invalid, the rest stays in force and the invalid provision is replaced by a
   valid one that comes closest to its purpose.
3. This agreement, the Confirmation and the privacy statement are the whole agreement between you and the Author on
   SQL Blame.
