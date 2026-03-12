# Mira: Costing & Pricing Strategy

## 1. Your Infrastructure Cost (What YOU Pay)

### Option A: Buy Hardware (One-Time CAPEX)

| Item | Cost (₹) | Frequency |
|---|---|---|
| **Agent Server** (used RTX 3090 24GB, 32GB RAM, 8-core CPU) | ₹80,000 - ₹1,00,000 | One-time |
| **Agent Server** (new RTX 4090 24GB build) | ₹1,50,000 - ₹2,00,000 | One-time |
| Electricity (~300W server, 24/7) | ₹2,000 - ₹3,000 | /month |
| Internet (static IP for WebSocket) | ₹1,000 - ₹2,000 | /month |
| Your existing VPS (FusionPBX) | Already paying | /month |
| **Total (used 3090 build)** | **₹80K upfront + ₹4K/month** | |

### Option B: Rent Cloud GPU (Recurring OPEX)

| Provider | GPU | Cost/hr | Cost/month (24/7) |
|---|---|---|---|
| **RunPod** | RTX 4090 | ~₹33/hr | ~₹24,000/month |
| **Vast.ai** | RTX 3090 | ~₹17/hr | ~₹12,000/month |
| **Lambda** | A10G | ~₹50/hr | ~₹36,000/month |

> **Verdict:** If you plan to serve 3+ doctors, **buying hardware pays for itself in 3-4 months** vs renting.

---

## 2. Per-Doctor Marginal Cost (What Each New Doctor Costs You)

This is the beauty of the multi-tenant architecture — adding a new doctor is almost **free.**

| Item | Cost per New Doctor |
|---|---|
| Database row (PostgreSQL) | ₹0 |
| System prompt template | ₹0 |
| Google Calendar API | ₹0 (free tier) |
| WhatsApp via Baileys | ₹0 |
| DID / Phone Number on PBX | ₹0 - ₹200/month (depends on your SIP provider) |
| GPU compute (shared across all doctors) | ₹0 marginal |
| **Total marginal cost per doctor** | **~₹0 - ₹200/month** |

One GPU serves **all** doctors simultaneously because calls don't overlap on the same millisecond — the LLM handles them sequentially/queued.

---

## 3. What a Human Receptionist Costs (Your Competition)

| Item | Cost |
|---|---|
| Full-time receptionist salary (Tier 2/3 city) | ₹8,000 - ₹15,000/month |
| Full-time receptionist salary (Metro city) | ₹12,000 - ₹20,000/month |
| Works only 8-10 hours/day | ❌ |
| Takes leaves, holidays | ❌ |
| Can handle only 1 call at a time | ❌ |

**Mira works 24/7/365, never takes a leave, and costs the doctor a fraction of this.**

---

## 4. Recommended Pricing (What YOU Charge Doctors)

### Pricing Tiers

| Plan | Price/month | What's Included |
|---|---|---|
| **Basic** | ₹2,999/month | Mira AI receptionist, appointment booking, 24/7 availability |
| **Pro** | ₹4,999/month | Basic + WhatsApp notifications + call recordings + monthly report |
| **Premium** | ₹7,999/month | Pro + multilingual (Hindi/English/Regional) + priority support |

### Annual Plans (incentivize lock-in)
| Plan | Monthly | Annual (20% discount) |
|---|---|---|
| Basic | ₹2,999 | ₹28,800/year (₹2,400/mo effective) |
| Pro | ₹4,999 | ₹48,000/year (₹4,000/mo effective) |
| Premium | ₹7,999 | ₹76,800/year (₹6,400/mo effective) |

---

## 5. Profitability Analysis

### Scenario: 10 Doctors on Basic Plan (₹2,999/month each)

| | Monthly |
|---|---|
| **Revenue** (10 × ₹2,999) | ₹29,990 |
| **Costs:** | |
| Server electricity | -₹3,000 |
| Internet | -₹2,000 |
| DID numbers (10 × ₹200) | -₹2,000 |
| **Net Profit** | **₹22,990/month** |

### Breakeven Analysis (if buying hardware)

| Hardware Cost | Monthly Profit | Breakeven |
|---|---|---|
| ₹80,000 (used 3090) | ₹22,990 | **~3.5 months** |
| ₹1,50,000 (new 4090) | ₹22,990 | **~6.5 months** |

### Scale Projection

| Doctors | Monthly Revenue | Monthly Cost | Monthly Profit |
|---|---|---|---|
| 3 | ₹9,000 | ₹5,600 | ₹3,400 |
| 5 | ₹15,000 | ₹6,000 | ₹9,000 |
| 10 | ₹30,000 | ₹7,000 | ₹23,000 |
| 25 | ₹75,000 | ₹8,000 | ₹67,000 |
| 50 | ₹1,50,000 | ₹10,000 | ₹1,40,000 |

> At 50 doctors, you'd need a second GPU server (~₹1L), but you'd be making ₹1.4L/month profit.

---

## 6. Summary: Your Investment vs Return

| | Buy (3090) | Rent (Vast.ai) |
|---|---|---|
| **Upfront** | ₹80,000 | ₹0 |
| **Monthly fixed cost** | ₹5,000 | ₹17,000 |
| **Breakeven (3 doctors)** | Month 4 | Month 1 |
| **Profit at 10 doctors** | ₹23K/month | ₹11K/month |
| **Better for** | Long-term (3+ months) | Testing / MVP phase |

### My Recommendation
1. **Start with Vast.ai rental** (~₹12-17K/mo) to build and test Mira with your first 3 doctors.
2. Once you have 5+ paying doctors, **buy a used RTX 3090 build** (~₹80K) to maximize profit.
3. Scale to 50 doctors on one machine, then add a second server.
