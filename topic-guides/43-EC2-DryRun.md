## EC2 `-dry-run`
**Use:** Check if your IAM user has permission to perform an EC2 action **without actually doing it**. 
```bash
aws ec2 terminate-instances \
  --instance-ids i-1234567890abcdef0 \
  --dry-run
```
**Result:**
- `DryRunOperation` → permission **allowed** ✅
- `UnauthorizedOperation` → permission **denied** ❌
**Exam memory:**
> **EC2 permission check → ****`--dry-run`** 
