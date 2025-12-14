# Donation Tracking Stream Eng

## System Overview

Stream engine that matches donor contributions with organizational expenditures in chronological order, providing transparency through media-rich expenditure records and personalized donor dashboards.

## Tech Stack

- Frontend: Next.js 14 (App Router), TypeScript, Tailwind CSS
- Backend: Node.js, Express, TypeScript
- Database: PostgreSQL with Prisma ORM
- File Storage: AWS S3 or similar
- Authentication: NextAuth.js

## Database Schema

### Core Tables

**donors**
- id: string (UUID, primary key)
- email: string (unique)
- name: string
- created_at: timestamp
- updated_at: timestamp

**donations**
- id: string (UUID, primary key)
- donor_id: string (foreign key to donors)
- amount: decimal(10,2)
- donated_at: timestamp
- created_at: timestamp

**expenditures**
- id: string (UUID, primary key)
- amount: decimal(10,2)
- cause: string
- description: text
- spent_at: timestamp
- created_at: timestamp

**expenditure_media**
- id: string (UUID, primary key)
- expenditure_id: string (foreign key to expenditures)
- media_type: enum ('image', 'video')
- media_url: string
- created_at: timestamp

**donation_expenditure_matches**
- id: string (UUID, primary key)
- donation_id: string (foreign key to donations)
- expenditure_id: string (foreign key to expenditures)
- allocated_amount: decimal(10,2)
- created_at: timestamp

## File Structure

```
/project-root
├── /backend
│   ├── /src
│   │   ├── /config
│   │   │   └── database.ts
│   │   ├── /controllers
│   │   │   ├── donationController.ts
│   │   │   ├── expenditureController.ts
│   │   │   └── matchingController.ts
│   │   ├── /middleware
│   │   │   ├── auth.ts
│   │   │   └── validation.ts
│   │   ├── /models
│   │   │   └── (Prisma handles this)
│   │   ├── /routes
│   │   │   ├── donationRoutes.ts
│   │   │   ├── expenditureRoutes.ts
│   │   │   └── donorRoutes.ts
│   │   ├── /services
│   │   │   ├── matchingService.ts
│   │   │   ├── mediaService.ts
│   │   │   └── donationService.ts
│   │   ├── /types
│   │   │   └── index.ts
│   │   └── server.ts
│   ├── prisma
│   │   └── schema.prisma
│   └── package.json
│
├── /frontend
│   ├── /src
│   │   ├── /app
│   │   │   ├── /admin
│   │   │   │   ├── /expenditures
│   │   │   │   │   ├── page.tsx
│   │   │   │   │   └── /new
│   │   │   │   │       └── page.tsx
│   │   │   │   └── /donations
│   │   │   │       └── page.tsx
│   │   │   ├── /donor
│   │   │   │   ├── /dashboard
│   │   │   │   │   └── page.tsx
│   │   │   │   └── /login
│   │   │   │       └── page.tsx
│   │   │   └── layout.tsx
│   │   ├── /components
│   │   │   ├── /admin
│   │   │   │   ├── ExpenditureForm.tsx
│   │   │   │   ├── DonationForm.tsx
│   │   │   │   └── MediaUpload.tsx
│   │   │   ├── /donor
│   │   │   │   ├── ContributionCard.tsx
│   │   │   │   ├── ExpenditureDetail.tsx
│   │   │   │   └── DonorStats.tsx
│   │   │   └── /shared
│   │   │       └── MediaGallery.tsx
│   │   ├── /lib
│   │   │   ├── api.ts
│   │   │   └── utils.ts
│   │   └── /types
│   │       └── index.ts
│   └── package.json
```

## TypeScript Interfaces

### Shared Types (/frontend/src/types/index.ts and /backend/src/types/index.ts)

```typescript
export interface Donor {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface Donation {
  id: string;
  donorId: string;
  amount: number;
  donatedAt: Date;
  createdAt: Date;
}

export interface Expenditure {
  id: string;
  amount: number;
  cause: string;
  description: string;
  spentAt: Date;
  createdAt: Date;
}

export interface ExpenditureMedia {
  id: string;
  expenditureId: string;
  mediaType: 'image' | 'video';
  mediaUrl: string;
  createdAt: Date;
}

export interface DonationExpenditureMatch {
  id: string;
  donationId: string;
  expenditureId: string;
  allocatedAmount: number;
  createdAt: Date;
}

export interface DonorContribution {
  expenditure: Expenditure;
  allocatedAmount: number;
  media: ExpenditureMedia[];
}

export interface DonorDashboardData {
  donor: Donor;
  totalDonated: number;
  contributions: DonorContribution[];
}
```

## Key Backend Files

### /backend/src/services/matchingService.ts

```typescript
import { PrismaClient } from '@prisma/client';
import { Donation, Expenditure, DonationExpenditureMatch } from '../types';

const prisma = new PrismaClient();

export class MatchingService {
  async matchDonationsToExpenditures(): Promise<void> {
    // Get all unmatched donations ordered by time
    const donations = await prisma.donation.findMany({
      where: {
        matches: {
          none: {}
        }
      },
      orderBy: {
        donatedAt: 'asc'
      }
    });

    // Get all expenditures that still need funding
    const expenditures = await prisma.expenditure.findMany({
      orderBy: {
        spentAt: 'asc'
      }
    });

    for (const expenditure of expenditures) {
      await this.matchExpenditureWithDonations(expenditure);
    }
  }

  private async matchExpenditureWithDonations(
    expenditure: Expenditure
  ): Promise<void> {
    let remainingAmount = expenditure.amount;

    // Get existing matches for this expenditure
    const existingMatches = await prisma.donationExpenditureMatch.findMany({
      where: {
        expenditureId: expenditure.id
      }
    });

    const totalAllocated = existingMatches.reduce(
      (sum, match) => sum + match.allocatedAmount.toNumber(),
      0
    );

    remainingAmount = expenditure.amount - totalAllocated;

    if (remainingAmount <= 0) return;

    // Find donations that occurred before or at the time of expenditure
    const eligibleDonations = await prisma.donation.findMany({
      where: {
        donatedAt: {
          lte: expenditure.spentAt
        }
      },
      orderBy: {
        donatedAt: 'asc'
      }
    });

    for (const donation of eligibleDonations) {
      if (remainingAmount <= 0) break;

      // Calculate how much of this donation is already allocated
      const donationMatches = await prisma.donationExpenditureMatch.findMany({
        where: {
          donationId: donation.id
        }
      });

      const allocatedFromDonation = donationMatches.reduce(
        (sum, match) => sum + match.allocatedAmount.toNumber(),
        0
      );

      const availableFromDonation = donation.amount - allocatedFromDonation;

      if (availableFromDonation <= 0) continue;

      const amountToAllocate = Math.min(availableFromDonation, remainingAmount);

      await prisma.donationExpenditureMatch.create({
        data: {
          donationId: donation.id,
          expenditureId: expenditure.id,
          allocatedAmount: amountToAllocate
        }
      });

      remainingAmount -= amountToAllocate;
    }
  }
}
```

### /backend/src/controllers/expenditureController.ts

```typescript
import { Request, Response } from 'express';
import { PrismaClient } from '@prisma/client';
import { MatchingService } from '../services/matchingService';
import { MediaService } from '../services/mediaService';

const prisma = new PrismaClient();
const matchingService = new MatchingService();
const mediaService = new MediaService();

export class ExpenditureController {
  async createExpenditure(req: Request, res: Response): Promise<void> {
    try {
      const { amount, cause, description, spentAt } = req.body;
      const files = req.files as Express.Multer.File[];

      const expenditure = await prisma.expenditure.create({
        data: {
          amount: parseFloat(amount),
          cause,
          description,
          spentAt: new Date(spentAt)
        }
      });

      // Upload media files
      if (files && files.length > 0) {
        for (const file of files) {
          const mediaUrl = await mediaService.uploadFile(file);
          const mediaType = file.mimetype.startsWith('video') ? 'video' : 'image';

          await prisma.expenditureMedia.create({
            data: {
              expenditureId: expenditure.id,
              mediaType,
              mediaUrl
            }
          });
        }
      }

      // Trigger matching
      await matchingService.matchDonationsToExpenditures();

      res.status(201).json(expenditure);
    } catch (error) {
      res.status(500).json({ error: 'Failed to create expenditure' });
    }
  }

  async listExpenditures(req: Request, res: Response): Promise<void> {
    try {
      const expenditures = await prisma.expenditure.findMany({
        include: {
          media: true
        },
        orderBy: {
          spentAt: 'desc'
        }
      });

      res.json(expenditures);
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch expenditures' });
    }
  }
}
```

### /backend/src/controllers/donationController.ts

```typescript
import { Request, Response } from 'express';
import { PrismaClient } from '@prisma/client';
import { MatchingService } from '../services/matchingService';

const prisma = new PrismaClient();
const matchingService = new MatchingService();

export class DonationController {
  async createDonation(req: Request, res: Response): Promise<void> {
    try {
      const { donorId, amount, donatedAt } = req.body;

      const donation = await prisma.donation.create({
        data: {
          donorId,
          amount: parseFloat(amount),
          donatedAt: new Date(donatedAt)
        }
      });

      // Trigger matching
      await matchingService.matchDonationsToExpenditures();

      res.status(201).json(donation);
    } catch (error) {
      res.status(500).json({ error: 'Failed to create donation' });
    }
  }

  async listDonations(req: Request, res: Response): Promise<void> {
    try {
      const donations = await prisma.donation.findMany({
        include: {
          donor: true
        },
        orderBy: {
          donatedAt: 'desc'
        }
      });

      res.json(donations);
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch donations' });
    }
  }
}
```

### /backend/src/controllers/donorController.ts

```typescript
import { Request, Response } from 'express';
import { PrismaClient } from '@prisma/client';
import { DonorDashboardData } from '../types';

const prisma = new PrismaClient();

export class DonorController {
  async getDonorDashboard(req: Request, res: Response): Promise<void> {
    try {
      const donorId = req.params.donorId;

      const donor = await prisma.donor.findUnique({
        where: { id: donorId }
      });

      if (!donor) {
        res.status(404).json({ error: 'Donor not found' });
        return;
      }

      // Get all donations by this donor
      const donations = await prisma.donation.findMany({
        where: { donorId }
      });

      const totalDonated = donations.reduce((sum, d) => sum + d.amount, 0);

      // Get all matches for this donor's donations
      const matches = await prisma.donationExpenditureMatch.findMany({
        where: {
          donationId: {
            in: donations.map(d => d.id)
          }
        },
        include: {
          expenditure: {
            include: {
              media: true
            }
          }
        }
      });

      // Group by expenditure
      const contributionsMap = new Map();

      for (const match of matches) {
        const expenditureId = match.expenditureId;

        if (!contributionsMap.has(expenditureId)) {
          contributionsMap.set(expenditureId, {
            expenditure: match.expenditure,
            allocatedAmount: 0,
            media: match.expenditure.media
          });
        }

        const contribution = contributionsMap.get(expenditureId);
        contribution.allocatedAmount += match.allocatedAmount.toNumber();
      }

      const contributions = Array.from(contributionsMap.values());

      const dashboardData: DonorDashboardData = {
        donor,
        totalDonated,
        contributions
      };

      res.json(dashboardData);
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch donor dashboard' });
    }
  }
}
```

## Key Frontend Components

### /frontend/src/app/donor/dashboard/page.tsx

```typescript
'use client';

import { useEffect, useState } from 'react';
import { useSession } from 'next-auth/react';
import { DonorDashboardData } from '@/types';
import ContributionCard from '@/components/donor/ContributionCard';
import DonorStats from '@/components/donor/DonorStats';

export default function DonorDashboard() {
  const { data: session } = useSession();
  const [dashboardData, setDashboardData] = useState<DonorDashboardData | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (session?.user?.id) {
      fetchDashboardData(session.user.id);
    }
  }, [session]);

  const fetchDashboardData = async (donorId: string) => {
    try {
      const response = await fetch(`/api/donors/${donorId}/dashboard`);
      const data = await response.json();
      setDashboardData(data);
    } catch (error) {
      console.error('Failed to fetch dashboard data', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading...</div>;
  if (!dashboardData) return <div>No data available</div>;

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Your Impact Dashboard</h1>
      
      <DonorStats
        totalDonated={dashboardData.totalDonated}
        contributionCount={dashboardData.contributions.length}
      />

      <div className="mt-8">
        <h2 className="text-2xl font-semibold mb-4">Your Contributions</h2>
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          {dashboardData.contributions.map((contribution) => (
            <ContributionCard
              key={contribution.expenditure.id}
              contribution={contribution}
            />
          ))}
        </div>
      </div>
    </div>
  );
}
```

### /frontend/src/components/donor/ContributionCard.tsx

```typescript
import { DonorContribution } from '@/types';
import MediaGallery from '@/components/shared/MediaGallery';

interface ContributionCardProps {
  contribution: DonorContribution;
}

export default function ContributionCard({ contribution }: ContributionCardProps) {
  const { expenditure, allocatedAmount, media } = contribution;

  return (
    <div className="border rounded-lg overflow-hidden shadow-lg">
      <MediaGallery media={media} />
      
      <div className="p-4">
        <h3 className="font-semibold text-lg mb-2">{expenditure.cause}</h3>
        <p className="text-gray-600 text-sm mb-4">{expenditure.description}</p>
        
        <div className="flex justify-between items-center">
          <div>
            <p className="text-xs text-gray-500">Your contribution</p>
            <p className="text-lg font-bold text-green-600">
              ${allocatedAmount.toFixed(2)}
            </p>
          </div>
          <div>
            <p className="text-xs text-gray-500">Total spent</p>
            <p className="text-sm font-semibold">
              ${expenditure.amount.toFixed(2)}
            </p>
          </div>
        </div>
        
        <p className="text-xs text-gray-400 mt-4">
          Spent on {new Date(expenditure.spentAt).toLocaleDateString()}
        </p>
      </div>
    </div>
  );
}
```

### /frontend/src/app/admin/expenditures/new/page.tsx

```typescript
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import ExpenditureForm from '@/components/admin/ExpenditureForm';

export default function NewExpenditure() {
  const router = useRouter();
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async (formData: FormData) => {
    setSubmitting(true);
    try {
      const response = await fetch('/api/expenditures', {
        method: 'POST',
        body: formData
      });

      if (response.ok) {
        router.push('/admin/expenditures');
      }
    } catch (error) {
      console.error('Failed to create expenditure', error);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Create New Expenditure</h1>
      <ExpenditureForm onSubmit={handleSubmit} submitting={submitting} />
    </div>
  );
}
```

## API Routes

### Backend Routes (/backend/src/routes/)

**donationRoutes.ts**
- POST /api/donations - Create donation
- GET /api/donations - List all donations

**expenditureRoutes.ts**
- POST /api/expenditures - Create expenditure with media
- GET /api/expenditures - List all expenditures

**donorRoutes.ts**
- GET /api/donors/:donorId/dashboard - Get donor dashboard data
- POST /api/donors - Create donor account

## Matching Algorithm Flow

1. When a new donation is created, it enters the pool of unallocated funds
2. When a new expenditure is created, the system looks for donations that occurred before or at the time of the expenditure
3. Donations are allocated in chronological order (FIFO)
4. If a donation is larger than an expenditure, the remainder stays unallocated for future expenditures
5. If an expenditure is larger than available donations, it remains partially funded until more donations arrive
6. The donation_expenditure_matches table tracks which portion of each donation went to which expenditure

## Deployment Considerations

**Database**: PostgreSQL on managed service (AWS RDS, Supabase, or similar)

**File Storage**: AWS S3 or Cloudflare R2 for media files

**Backend**: Deploy on Railway, Render, or AWS ECS

**Frontend**: Deploy on Vercel or Netlify

**Authentication**: NextAuth.js with email/password or OAuth providers

## Security Considerations

- Admin routes protected with role-based authentication
- Donor routes protected with user authentication
- File uploads validated for type and size
- SQL injection prevention via Prisma ORM
- Rate limiting on API endpoints
- HTTPS enforced in production
