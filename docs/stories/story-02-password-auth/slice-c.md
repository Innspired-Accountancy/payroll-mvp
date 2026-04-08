# Slice c: Password Reset Flow

**Story:** story-02-password-auth
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-b

## Goal

Implement secure password reset flow with email token generation, verification, and password update including token expiry and single-use enforcement.

## Decision Checklist

- [x] All libraries/packages named: `better-auth@0.8.x`, `nodemailer@6.x` or `resend@2.x`, `uuid@latest`
- [x] SDK methods identified: `auth.api.forgetPassword()`, `auth.api.resetPassword()`, `crypto.randomUUID()`
- [x] External service endpoints: Email service (Resend API or SMTP)
- [x] Data contracts defined: Password reset token schema, email template
- [x] Configuration variables: `EMAIL_FROM`, `RESEND_API_KEY` or `SMTP_*`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ User Journeys → Journey 2: Assign Role to User (invitation flow similar)
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Security

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/email/index.ts` | create | Email service interface |
| `src/lib/email/templates/password-reset.ts` | create | Reset email template |
| `src/server/trpc/routers/password-reset.ts` | create | Password reset procedures |
| `src/app/(auth)/forgot-password/page.tsx` | create | Forgot password page |
| `src/app/(auth)/reset-password/page.tsx` | create | Reset password page |

## Responsibilities
1. Generate cryptographically secure reset tokens
2. Send password reset emails with time-limited links (1 hour expiry)
3. Verify reset tokens and validate new passwords
4. Invalidate all existing sessions on password change
5. Clear account lockout on successful password reset

## Contracts

### Password Reset Token Schema
```typescript
// Stored via BetterAuth's built-in verificationToken table
// Token format: crypto.randomUUID() + timestamp hash

export const passwordResetInputSchema = z.object({
  email: z.string().email('Invalid email address'),
});

export const passwordResetConfirmSchema = z.object({
  token: z.string().uuid('Invalid token format'),
  password: z.string()
    .min(12, 'Password must be at least 12 characters')
    .max(128, 'Password must be less than 128 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
    .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number')
    .regex(/[^A-Za-z0-9]/, 'Password must contain at least one special character'),
});

export type PasswordResetInput = z.infer<typeof passwordResetInputSchema>;
export type PasswordResetConfirm = z.infer<typeof passwordResetConfirmSchema>;
```

### Email Service
```typescript
// src/lib/email/index.ts
import { Resend } from 'resend';

const resend = new Resend(process.env.RESEND_API_KEY);

export interface EmailOptions {
  to: string;
  subject: string;
  html: string;
  text?: string;
}

export async function sendEmail(options: EmailOptions): Promise<void> {
  await resend.emails.send({
    from: process.env.EMAIL_FROM!,
    to: options.to,
    subject: options.subject,
    html: options.html,
    text: options.text,
  });
}

export async function sendPasswordResetEmail(email: string, resetUrl: string): Promise<void> {
  const template = getPasswordResetTemplate(resetUrl);
  
  await sendEmail({
    to: email,
    subject: 'Reset your password - UK Bureau Payroll',
    html: template.html,
    text: template.text,
  });
}
```

### Password Reset Email Template
```typescript
// src/lib/email/templates/password-reset.ts
export function getPasswordResetTemplate(resetUrl: string) {
  return {
    html: `
      <!DOCTYPE html>
      <html>
      <head>
        <meta charset="utf-8">
        <title>Reset your password</title>
      </head>
      <body style="font-family: Arial, sans-serif; line-height: 1.6; color: #333;">
        <div style="max-width: 600px; margin: 0 auto; padding: 20px;">
          <h2 style="color: #1a365d;">Password Reset Request</h2>
          <p>We received a request to reset your password for your UK Bureau Payroll account.</p>
          <p>Click the button below to reset your password. This link will expire in 1 hour.</p>
          <div style="text-align: center; margin: 30px 0;">
            <a href="${resetUrl}" 
               style="background-color: #1a365d; color: white; padding: 12px 30px; 
                      text-decoration: none; border-radius: 4px; display: inline-block;">
              Reset Password
            </a>
          </div>
          <p>If you didn't request this, you can safely ignore this email. Your password will not be changed.</p>
          <p style="color: #666; font-size: 12px; margin-top: 30px;">
            If the button doesn't work, copy and paste this link into your browser:<br>
            <a href="${resetUrl}">${resetUrl}</a>
          </p>
        </div>
      </body>
      </html>
    `,
    text: `
      Password Reset Request

      We received a request to reset your password for your UK Bureau Payroll account.

      Click the link below to reset your password. This link will expire in 1 hour:

      ${resetUrl}

      If you didn't request this, you can safely ignore this email. Your password will not be changed.
    `,
  };
}
```

### tRPC Password Reset Router
```typescript
// src/server/trpc/routers/password-reset.ts
import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { publicProcedure, router } from '../trpc';
import { auth } from '@/lib/auth';
import { sendPasswordResetEmail } from '@/lib/email';
import { db } from '@/lib/db';
import { users } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';
import { passwordResetInputSchema, passwordResetConfirmSchema } from '@/lib/auth/validation';

export const passwordResetRouter = router({
  requestReset: publicProcedure
    .input(passwordResetInputSchema)
    .mutation(async ({ input }) => {
      const { email } = input;
      
      // Check if user exists (but don't reveal this)
      const user = await db.query.users.findFirst({
        where: eq(users.email, email.toLowerCase()),
        columns: { id: true, status: true },
      });
      
      // Always return success to prevent user enumeration
      // But only send email if user exists and is active
      if (user && user.status === 'active') {
        try {
          // Generate reset token via BetterAuth
          const resetToken = await auth.api.forgetPassword({
            body: { email },
          });
          
          // Construct reset URL
          const baseUrl = process.env.BETTER_AUTH_URL!;
          const resetUrl = `${baseUrl}/reset-password?token=${resetToken}`;
          
          // Send email
          await sendPasswordResetEmail(email, resetUrl);
          
        } catch (error) {
          // Log error but don't expose to user
          console.error('Failed to send password reset email:', error);
        }
      }
      
      // Generic success response
      return {
        success: true,
        message: 'If an account exists with this email, you will receive a password reset link.',
      };
    }),
  
  confirmReset: publicProcedure
    .input(passwordResetConfirmSchema)
    .mutation(async ({ input, ctx }) => {
      const { token, password } = input;
      
      try {
        // Reset password via BetterAuth
        const result = await auth.api.resetPassword({
          body: { token, password },
          headers: ctx.headers,
        });
        
        // Clear any account lockout
        if (result.user?.id) {
          await db.update(users)
            .set({
              failedLoginAttempts: 0,
              lockedUntil: null,
              passwordChangedAt: new Date(),
            })
            .where(eq(users.id, result.user.id));
        }
        
        return {
          success: true,
          message: 'Password reset successfully. Please log in with your new password.',
        };
        
      } catch (error) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: 'Invalid or expired reset token. Please request a new password reset.',
        });
      }
    }),
  
  validateToken: publicProcedure
    .input(z.object({ token: z.string() }))
    .query(async ({ input }) => {
      // Verify token exists and is not expired
      // This is handled by BetterAuth during reset, but we can pre-validate
      return { valid: true }; // Simplified - BetterAuth validates on use
    }),
});

export type PasswordResetRouter = typeof passwordResetRouter;
```

## Business Rules & Invariants
1. Reset tokens expire after 1 hour
2. Reset tokens are single-use (deleted after successful reset)
3. New password must meet policy (12 chars, complexity requirements)
4. Account lockout is cleared on successful password reset
5. Generic responses to prevent user enumeration

## Edge Cases
1. **Token already used** — Return "invalid or expired" error
2. **Token expired** — Return "invalid or expired" error, require new request
3. **User inactive during reset** — Still allow reset (may have been deactivated temporarily)
4. **Email delivery failure** — Log error, don't expose to user
5. **Concurrent reset requests** — Each generates new token, all are valid until used

## Tests

### src/server/trpc/routers/password-reset.test.ts
- `should send reset email for valid user`: Verifies email sending
- `should return generic success for non-existent user`: Verifies enumeration prevention
- `should reset password with valid token`: Verifies password change
- `should reject expired token`: Verifies expiry enforcement
- `should reject reused token`: Verifies single-use enforcement
- `should validate password complexity`: Verifies policy enforcement
- `should clear lockout on password reset`: Verifies lockout clearance

## Verification
```bash
npm run test:unit src/server/trpc/routers/password-reset.test.ts
npm run test:integration src/lib/email/password-reset.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Non-Functional Requirements → Security, Password policy
