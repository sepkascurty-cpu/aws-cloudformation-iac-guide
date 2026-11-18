# Quiz — AWS CloudFormation

Test yourself: 5 questions, answers hidden below. No peeking!

---

## Questions

**1.** You run `UpdateStack` to change your RDS instance class from `db.t3.micro` to `db.t3.small`, and CloudFormation reports the update will **replace** the database. What does "replacement" actually mean here, and which two template settings decide whether your data survives it?

**2.** A teammate edited an EC2 security group rule directly in the AWS console. The stack still shows `CREATE_COMPLETE` and nothing looks broken. How do you prove the stack has drifted, what does the drift report tell you, and what is the correct way to fix it?

**3.** Your template hard-codes `ImageId: ami-0c02fb55956c7d316` for the launch template. The stack deploys fine in `us-east-1` but fails in `ap-southeast-1` with "InvalidAMIID.NotFound". Name **two** template-native ways to fix this so the same template works in any region.

**4.** Two teams each deploy a stack that exports a value named `VpcId`. The second deploy fails. Why — and what naming rule prevents this?

**5.** You update a template, changing only a tag on the Auto Scaling group — but the change set shows the RDS instance will be **replaced**. Give **two** plausible reasons the database is in the replacement list even though you didn't touch it, and name the one command you should have run before executing the update.

---

## Answers

<details>
<summary><strong>Click to reveal answers</strong></summary>

**1.** Replacement means CloudFormation creates a **brand-new** DB instance and deletes the old one — it is not an in-place resize. (Some properties, like instance class on RDS, actually update in place; but engine version changes and certain property edits do force replacement, and the change set is the only reliable way to know.) Whether data survives depends on: (a) **DeletionPolicy** — `Snapshot` makes CloudFormation take a final snapshot before deleting the old instance; `Retain` keeps the old instance alive (orphaned, still billable); `Delete` (default) drops it with your data; and (b) **UpdateReplacePolicy** — the same idea, applied specifically when replacement happens during an update. Set both to `Snapshot` on every stateful resource.

**2.** Run drift detection: `aws cloudformation detect-stack-drift --stack-name <name>`, then `describe-stack-resource-drifts`. The report lists each resource with `StackResourceDriftStatus` (`IN_SYNC` vs `MODIFIED`) and the exact property that differs (`ExpectedValue` vs `ActualValue`). The correct fix is to **update the template** to match the intended rule and deploy it — never edit in the console again. (Re-applying the stack without changing the template also reverts the drift, but it silently discards a change someone may have made deliberately — so capture it in the template first.)

**3.** (a) A **Mappings** table keyed by region, looked up with `!FindInMap [RegionToAmi, !Ref "AWS::Region", Ami]` — the template in this repo does exactly this. (b) An **SSM parameter** for the latest AMI, e.g. type `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` with default `/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2` — AWS resolves it at deploy time, always current, never region-locked.

**4.** Export names are **unique per region**. Two stacks can't export the same name in the same region, so the second deploy fails with a validation error. Rule: prefix every export with the stack name, e.g. `!Sub "${AWS::StackName}-VpcId"`, so exports can never collide.

**5.** Two plausible reasons: (a) the update changed an **immutable property** of the database indirectly — e.g. you edited the `DBSubnetGroup` or the security group the DB references in a way that forces replacement, or the parameter feeding the DB (engine version, storage type) changed; (b) the **change set was generated from a different template version** than you think — e.g. a teammate's edit to the same file, or you ran the update against a stale local copy. Either way, the command you should have run first is `create-change-set` + `describe-change-set` — the change set shows `Replacement: True` on the RDS instance before anything is executed, so you can stop, investigate, and never execute a surprise database replacement.

</details>
