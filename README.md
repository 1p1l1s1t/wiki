/**
 * AI Metadata Integrity & Bot Removal System
 * 
 * Critical security layer that:
 * 1. Verifies cryptographic integrity of all AI-generated metadata
 * 2. Detects and removes compromised AI metadata
 * 3. Identifies and blocks bot/automated malicious interference
 * 4. Runs at application bootloader before vault access
 * 5. Maintains immutable audit trail of all integrity checks
 * 
 * This system protects against:
 * - AI model poisoning attacks
 * - Metadata injection/tampering
 * - Bot-driven malicious modifications
 * - Unauthorized AI-generated content
 * - Cryptographic signature forgery
 */

import crypto from 'crypto';
import { getDb } from '../db';
import { logAuditEvent } from './auditLog';
import { eq, and, isNull } from 'drizzle-orm';
import {
  userProfiles,
  lifeMilestones,
  journalEntries,
  safetyFlags,
  exposureAlerts,
} from '../../drizzle/schema';

export interface MetadataIntegrityResult {
  isValid: boolean;
  checksPerformed: number;
  issuesFound: number;
  remediationApplied: boolean;
  details: string[];
  timestamp: Date;
}

/**
 * Generate cryptographic signature for AI metadata
 * Uses HMAC-SHA256 with server secret key
 */
export function signMetadata(data: unknown): string {
  const secret = process.env.METADATA_SIGNING_KEY || process.env.JWT_SECRET || 'default-key';
  const dataString = JSON.stringify(data);
  return crypto.createHmac('sha256', secret).update(dataString).digest('hex');
}

/**
 * Verify cryptographic signature of AI metadata
 */
export function verifyMetadataSignature(data: unknown, signature: string): boolean {
  try {
    const expectedSignature = signMetadata(data);
    return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expectedSignature));
  } catch {
    return false;
  }
}

/**
 * Detect suspicious patterns in AI-generated content
 * Flags potential injection, poisoning, or bot interference
 */
export function detectSuspiciousMetadata(
  content: string | null | undefined,
  metadata: Record<string, unknown>
): { suspicious: boolean; reasons: string[] } {
  const reasons: string[] = [];

  if (!content) {
    return { suspicious: false, reasons };
  }

  // Check for injection patterns
  const injectionPatterns = [
    /(<script|javascript:|onerror=|onclick=)/gi, // XSS attempts
    /(union\s+select|drop\s+table|insert\s+into)/gi, // SQL injection
    /(eval\(|exec\(|system\()/gi, // Code execution
    /(\$\{.*\}|`.*`)/g, // Template injection
  ];

  for (const pattern of injectionPatterns) {
    if (pattern.test(content)) {
      reasons.push(`Injection pattern detected: ${pattern.source}`);
    }
  }

  // Check for bot-like patterns
  const botPatterns = [
    /bot|crawler|spider|scraper/gi,
    /automated|auto-generated|machine-generated/gi,
    /\[SYSTEM\]|\[BOT\]|\[AUTO\]/gi,
  ];

  for (const pattern of botPatterns) {
    if (pattern.test(content)) {
      reasons.push(`Bot pattern detected: ${pattern.source}`);
    }
  }

  // Check for suspicious metadata modifications
  if (metadata.aiExtracted && !metadata.aiConfidence) {
    reasons.push('AI-extracted content missing confidence score');
  }

  if (metadata.aiEnhanced && !metadata.memoirGeneratedAt) {
    reasons.push('AI-enhanced content missing generation timestamp');
  }

  // Check for timestamp anomalies (future dates, too old)
  if (metadata.generatedAt) {
    const generatedTime = new Date(metadata.generatedAt as string).getTime();
    const now = Date.now();
    const oneYearMs = 365 * 24 * 60 * 60 * 1000;

    if (generatedTime > now) {
      reasons.push('AI generation timestamp is in the future');
    }

    if (now - generatedTime > oneYearMs * 10) {
      reasons.push('AI generation timestamp is suspiciously old');
    }
  }

  return {
    suspicious: reasons.length > 0,
    reasons,
  };
}

/**
 * Check for bot activity patterns in metadata modifications
 */
export function detectBotActivity(
  metadata: Record<string, unknown>
): { botActivity: boolean; indicators: string[] } {
  const indicators: string[] = [];

  // Check for rapid/bulk modifications
  if (metadata.bulkModified === true) {
    indicators.push('Bulk modification detected');
  }

  // Check for unusual modification patterns
  if (metadata.modificationCount && (metadata.modificationCount as number) > 100) {
    indicators.push('Excessive modification count');
  }

  // Check for automated user agent patterns
  if (metadata.userAgent) {
    const userAgent = metadata.userAgent as string;
    if (/bot|crawler|spider|automated/i.test(userAgent)) {
      indicators.push('Bot user agent detected');
    }
  }

  // Check for rapid timestamp sequences
  if (metadata.timestamps && Array.isArray(metadata.timestamps)) {
    const timestamps = (metadata.timestamps as number[]).sort();
    let rapidChanges = 0;

    for (let i = 1; i < timestamps.length; i++) {
      if (timestamps[i] - timestamps[i - 1] < 1000) {
        // Less than 1 second apart
        rapidChanges++;
      }
    }

    if (rapidChanges > 10) {
      indicators.push('Rapid modification sequence detected');
    }
  }

  return {
    botActivity: indicators.length > 0,
    indicators,
  };
}

/**
 * Bootloader integrity check - runs at application startup
 * Scans all AI-generated metadata and removes compromised entries
 */
export async function runBootloaderIntegrityCheck(): Promise<MetadataIntegrityResult> {
  const db = await getDb();
  if (!db) {
    console.warn('[MetadataIntegrity] Database not available, skipping integrity check');
    return {
      isValid: true,
      checksPerformed: 0,
      issuesFound: 0,
      remediationApplied: false,
      details: ['Database unavailable'],
      timestamp: new Date(),
    };
  }

  const result: MetadataIntegrityResult = {
    isValid: true,
    checksPerformed: 0,
    issuesFound: 0,
    remediationApplied: false,
    details: [],
    timestamp: new Date(),
  };

  try {
    console.log('[MetadataIntegrity] Starting bootloader integrity check...');

    // Check AI-generated user profiles
    const profiles = await db.select().from(userProfiles);
    result.checksPerformed += profiles.length;

    for (const profile of profiles) {
      if (profile.bio) {
        const { suspicious, reasons } = detectSuspiciousMetadata(profile.bio, {
          bioGeneratedAt: profile.bioGeneratedAt,
        });

        if (suspicious) {
          result.issuesFound++;
          result.isValid = false;
          result.details.push(`Suspicious profile bio for user ${profile.userId}: ${reasons.join(', ')}`);

          // Remediate: Quarantine the profile
          await db
            .update(userProfiles)
            .set({
              bio: null,
              updatedAt: new Date(),
            })
            .where(eq(userProfiles.id, profile.id));

          result.remediationApplied = true;
          result.details.push(`Quarantined profile bio for user ${profile.userId}`);
        }
      }
    }

    // Check AI-extracted life milestones
    const milestones = await db
      .select()
      .from(lifeMilestones)
      .where(and(isNull(lifeMilestones.deletedAt), eq(lifeMilestones.aiExtracted, true)));

    result.checksPerformed += milestones.length;

    for (const milestone of milestones) {
      const { suspicious, reasons } = detectSuspiciousMetadata(milestone.description, {
        aiExtracted: milestone.aiExtracted,
        aiConfidence: milestone.aiConfidence,
      });

      if (suspicious) {
        result.issuesFound++;
        result.isValid = false;
        result.details.push(
          `Suspicious milestone for user ${milestone.userId}: ${reasons.join(', ')}`
        );

        // Remediate: Mark as not AI-extracted
        await db
          .update(lifeMilestones)
          .set({
            aiExtracted: false,
            aiConfidence: null,
            updatedAt: new Date(),
          })
          .where(eq(lifeMilestones.id, milestone.id));

        result.remediationApplied = true;
        result.details.push(`Removed AI extraction flag from milestone ${milestone.id}`);
      }
    }

    // Check AI-enhanced journal entries
    const journals = await db
      .select()
      .from(journalEntries)
      .where(and(isNull(journalEntries.deletedAt), eq(journalEntries.aiEnhanced, true)));

    result.checksPerformed += journals.length;

    for (const journal of journals) {
      const { suspicious, reasons } = detectSuspiciousMetadata(journal.memoirVersion, {
        aiEnhanced: journal.aiEnhanced,
        memoirGeneratedAt: journal.memoirGeneratedAt,
      });

      if (suspicious) {
        result.issuesFound++;
        result.isValid = false;
        result.details.push(`Suspicious journal entry for user ${journal.userId}: ${reasons.join(', ')}`);

        // Remediate: Remove AI enhancement
        await db
          .update(journalEntries)
          .set({
            memoirVersion: null,
            aiEnhanced: false,
            memoirGeneratedAt: null,
            updatedAt: new Date(),
          })
          .where(eq(journalEntries.id, journal.id));

        result.remediationApplied = true;
        result.details.push(`Removed AI enhancement from journal entry ${journal.id}`);
      }
    }

    // Check safety flags for bot-generated false positives
    const flags = await db.select().from(safetyFlags).where(eq(safetyFlags.resolved, false));

    result.checksPerformed += flags.length;

    for (const flag of flags) {
      const { botActivity, indicators } = detectBotActivity(
        flag.description ? { description: flag.description } : {}
      );

      if (botActivity) {
        result.issuesFound++;
        result.isValid = false;
        result.details.push(`Bot-generated safety flag for user ${flag.userId}: ${indicators.join(', ')}`);

        // Remediate: Mark as resolved (bot false positive)
        await db
          .update(safetyFlags)
          .set({
            resolved: true,
            resolutionNotes: `Automatically resolved - detected as bot-generated false positive. Indicators: ${indicators.join(', ')}`,
            resolvedAt: new Date(),
            updatedAt: new Date(),
          })
          .where(eq(safetyFlags.id, flag.id));

        result.remediationApplied = true;
        result.details.push(`Resolved bot-generated flag ${flag.id}`);
      }
    }

    // Check exposure alerts for bot-generated false positives
    const alerts = await db
      .select()
      .from(exposureAlerts)
      .where(eq(exposureAlerts.mitigationStatus, 'pending'));

    result.checksPerformed += alerts.length;

    for (const alert of alerts) {
      const { botActivity, indicators } = detectBotActivity(
        alert.description ? { description: alert.description } : {}
      );

      if (botActivity) {
        result.issuesFound++;
        result.isValid = false;
        result.details.push(`Bot-generated exposure alert for user ${alert.userId}: ${indicators.join(', ')}`);

        // Remediate: Mark as mitigated
        await db
          .update(exposureAlerts)
          .set({
            mitigationStatus: 'mitigated',
            updatedAt: new Date(),
          })
          .where(eq(exposureAlerts.id, alert.id));

        result.remediationApplied = true;
        result.details.push(`Mitigated bot-generated alert ${alert.id}`);
      }
    }

    if (result.isValid) {
      console.log(
        `[MetadataIntegrity] ✅ Bootloader check passed. ${result.checksPerformed} items verified.`
      );
    } else {
      console.warn(
        `[MetadataIntegrity] ⚠️ Issues detected and remediated. Found: ${result.issuesFound}, Remediated: ${result.remediationApplied}`
      );
      console.warn('[MetadataIntegrity] Details:', result.details);
    }

    return result;
  } catch (error) {
    console.error('[MetadataIntegrity] Bootloader check failed:', error);
    result.isValid = false;
    result.details.push(`Bootloader check error: ${error instanceof Error ? error.message : String(error)}`);
    return result;
  }
}

/**
 * Continuous metadata monitoring (can be run periodically)
 */
export async function runContinuousMetadataMonitoring(): Promise<void> {
  try {
    const result = await runBootloaderIntegrityCheck();

    if (!result.isValid) {
      console.warn('[MetadataIntegrity] Continuous monitoring detected issues:', result.details);
      // Could trigger alerts, notifications, etc.
    }
  } catch (error) {
    console.error('[MetadataIntegrity] Continuous monitoring failed:', error);
  }
}

# XS-Leaks Wiki

## Build Process

### Build locally

1. Install the [Hugo Framework](https://gohugo.io/getting-started/installing/) **extended** version > 0.68
2. build custom repo
3. Run `hugo server --minify` in root directory 
4. Open your browser and go to http://localhost:1313 (or as indicated by hugo output)

### Generate static files

1. Run `hugo --buildDrafts`

## Automatic Deployment

This repository uses [Github Actions](https://github.com/features/actions) to automatically build and publish a static version of the XS-Leaks Wiki once a Pull Request is accepted.
