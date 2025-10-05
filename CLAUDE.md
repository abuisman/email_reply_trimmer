# Email Reply Trimmer - Maintenance Guide for Claude

This document explains how the `email_reply_trimmer` gem works internally and provides detailed instructions for adding new email patterns and test cases.

## Table of Contents

1. [How the Gem Works](#how-the-gem-works)
2. [Architecture Overview](#architecture-overview)
3. [Pattern Matching System](#pattern-matching-system)
4. [Adding New Test Cases](#adding-new-test-cases)
5. [Adding New Patterns](#adding-new-patterns)
6. [Testing Workflow](#testing-workflow)
7. [Common Email Patterns](#common-email-patterns)

---

## How the Gem Works

The `EmailReplyTrimmer` gem processes plain text emails to remove quoted replies, signatures, and embedded email markers, keeping only the new content written by the sender.

### High-Level Process

1. **Preprocessing** - Normalize line endings, remove PGP markers, clean up quote formats
2. **Code Block Hoisting** - Temporarily replace code blocks (```...```) with tokens to protect them
3. **Line-by-Line Analysis** - Each line is classified into one of these categories:
   - `d` - Delimiter (horizontal rules, etc.)
   - `b` - Embedded email marker (forwarded message headers)
   - `e` - Empty line
   - `h` - Email header (From:, To:, Sent:, etc.)
   - `q` - Quote (lines starting with >)
   - `s` - Signature (mobile/app signatures)
   - `t` - Text (actual content)

4. **Pattern Matching** - The sequence of characters (e.g., "tttteebhhhhtttt") is analyzed using regex patterns
5. **Trimming Logic** - Based on the pattern, lines are removed to keep only new content
6. **Code Block Restoration** - Protected code blocks are re-injected into the final output

### Example Flow

**Input Email:**
```
Thanks for the update!

Sent from my iPhone

From: John <john@example.com>
Sent: Monday, Oct 1, 2025 3:45 PM
To: Jane <jane@example.com>
Subject: Project Update

Hi Jane, here's the status...
```

**Line Classification:**
```
t  - "Thanks for the update!"
e  - (empty line)
s  - "Sent from my iPhone"
e  - (empty line)
h  - "From: John <john@example.com>"
h  - "Sent: Monday, Oct 1, 2025 3:45 PM"
h  - "To: Jane <jane@example.com>"
h  - "Subject: Project Update"
e  - (empty line)
t  - "Hi Jane, here's the status..."
```

**Pattern:** `tesehhhhht`

**Trimming Logic:** The algorithm detects 3+ consecutive email headers (`hhh`) and takes everything before them.

**Output:**
```
Thanks for the update!
```

---

## Architecture Overview

### File Structure

```
lib/
  email_reply_trimmer.rb                 # Main algorithm and orchestration
  email_reply_trimmer/
    delimiter_matcher.rb                 # Matches horizontal rules/delimiters
    email_header_matcher.rb              # Matches email headers (From:, To:, etc.)
    embedded_email_matcher.rb            # Matches forwarded/replied email markers
    empty_line_matcher.rb                # Matches empty/whitespace lines
    quote_matcher.rb                     # Matches quoted lines (>)
    signature_matcher.rb                 # Matches mobile/app signatures

test/
  emails/           # Input emails to test
  trimmed/          # Expected trimmed output
  elided/           # Expected removed content
  test_email_reply_trimmer.rb           # Test runner
```

### Matcher Files

Each matcher file contains:
- **Regex patterns** for different languages/formats
- A `match?(line)` method that returns true if the line matches any pattern

---

## Pattern Matching System

### 1. Delimiter Matcher (`delimiter_matcher.rb`)

Matches horizontal rules and visual separators:
- `________` (underscores)
- `--------` (dashes)
- `========` (equals)
- `********` (asterisks)

**When to add patterns here:** If emails use visual separators that should mark the end of new content.

### 2. Signature Matcher (`signature_matcher.rb`)

Matches mobile and app signatures that should be removed from the trimmed output:

```ruby
SIGNATURE_REGEXES = [
  # English
  /^[[:blank:]]*Get Outlook for /i,
  /^[[:blank:]]*Sent from Outlook for /i,
  /^[[:blank:]]*Sent from my iPhone/i,
  # Dutch
  /^[[:blank:]]*Verzonden vanuit /i,
  /^[[:blank:]]*Verstuurd vanaf mijn /i,
  # etc...
]
```

**When to add patterns here:** When you find mobile/app signatures that should be kept WITH the user's content (not removed as part of the quoted reply).

**Important:** These signatures are REMOVED from the final output. They mark the boundary but are not kept.

### 3. Email Header Matcher (`email_header_matcher.rb`)

Matches email metadata headers in various languages:

#### Date Headers (with timestamps):
```ruby
EMAIL_HEADERS_WITH_DATE_MARKERS = [
  ["Sent", "Date"],           # English
  ["Verzonden"],              # Dutch
  ["Gesendet"],               # German
  ["Envoyé"],                 # French
  ["Enviado"],                # Spanish
  ["Data"],                   # Italian
  # etc...
]
```

#### Text Headers (with names/subjects):
```ruby
EMAIL_HEADERS_WITH_TEXT_MARKERS = [
  ["From", "To", "Cc", "Subject"],          # English
  ["Van", "Aan", "Onderwerp"],              # Dutch
  ["Von", "An", "Betreff"],                 # German
  ["De", "À", "Objet"],                     # French
  # etc...
]
```

**When to add patterns here:** When Outlook/email clients in new languages use different header field names.

**Example patterns matched:**
- `From: John Smith <john@example.com>`
- `Van: Jan de Vries <jan@example.nl>`
- `Sent: Monday, October 1, 2025 3:45 PM`
- `Verzonden: maandag 1 oktober 2025 15:45`

### 4. Embedded Email Matcher (`embedded_email_matcher.rb`)

The most complex matcher - handles various "on [date] [someone] wrote:" patterns:

#### Pattern Categories:

**A. ON_DATE_SOMEONE_WROTE** - "On [date], [person] wrote:"
```ruby
# English: On Wed, Sep 25, 2013, at 03:57 PM, jorge_castro wrote:
/^[[:blank:]<>-]*(On|At) (?:(?!\b(?>On|wrote|writes)\b).)+?(wrote|writes)[[:blank:].:>-]*$/im

# Dutch: Op 24 aug. 2013 om 16:48 heeft ven88 het volgende geschreven:
/^[[:blank:]<>-]*Op (?:(?!\b(?>Op|het\svolgende\sgeschreven)\b).)+?het\svolgende\sgeschreven[[:blank:].:>-]*$/im
```

**B. ON_DATE_WROTE_SOMEONE** - "On [date] [person] wrote:"
```ruby
# Dutch: Op 10 dec. 2015 18:35 schreef "Arpit Jalan":
# German: Am 18.09.2013 um 16:24 schrieb codinghorror:
ON_DATE_WROTE_SOMEONE_MARKERS = [
  ["Op", "schreef"],   # Dutch
  ["Am", "schrieb"],   # German
]
```

**C. FORWARDED_EMAIL** - Forward markers
```ruby
FORWARDED_EMAIL_REGEXES = [
  /^[[:blank:]>]*Begin forwarded message:/i,
  /^[[:blank:]>*]*-{2,}[[:blank:]]*(Forwarded|Original|Reply) Message[[:blank:]]*-{2,}/i,
  /^[[:blank:]>*]*-{2,}[[:blank:]]*Ursprüngliche Nachricht[[:blank:]]*-{2,}/i,  # German
]
```

**When to add patterns here:** When you find "person X wrote on date Y" patterns that aren't being detected.

### 5. Quote Matcher (`quote_matcher.rb`)

Matches lines starting with `>` (standard email quote format).

**When to add patterns here:** Rarely needed - quote format is standardized.

### 6. Empty Line Matcher (`empty_line_matcher.rb`)

Matches blank lines or lines with only whitespace.

**When to add patterns here:** Never - this is complete.

---

## Adding New Test Cases

### Step-by-Step Process

#### 1. Identify the Problem Email

When you find an email that's not being trimmed correctly, save it as a test case.

#### 2. Anonymize the Email

**Replace:**
- Real names → Generic names (John Smith, Jane Doe, etc.)
- Real email addresses → example.com addresses
- Company names → Generic names (Acme, TechCorp, etc.)
- Phone numbers → Generic numbers
- Sensitive content → Generic content

**Keep:**
- The exact structure and formatting
- Language-specific keywords
- Email header format
- Signature patterns
- All whitespace and line breaks

#### 3. Create Three Test Files

For a test named `outlook_pattern_xyz`:

**a. Input email** - `test/emails/outlook_pattern_xyz.txt`
```
The complete email as received
```

**b. Expected trimmed output** - `test/trimmed/outlook_pattern_xyz.txt`
```
Only the new content you want to keep
(excluding quoted replies, but including signatures)
```

**c. Expected elided content** - `test/elided/outlook_pattern_xyz.txt`
```
Everything that should be removed
(the quoted/forwarded content)
```

#### 4. Run the Tests

```bash
rake test
```

If the test fails, it will show:
- What the gem currently produces
- What you expected
- The difference between them

#### 5. Example Test Case

**Input:** `test/emails/outlook_dutch_ios.txt`
```
Bedankt voor je bericht.

Met vriendelijke groet

Jan de Vries
Manager Operations

Verzonden vanuit Outlook voor iOS

Van: Team Support <support@example.com>
Verzonden: dinsdag, september 30, 2025 5:17 PM
Aan: jan.devries@company.nl
Onderwerp: RE: Project aanvraag

Beste Jan,
Graag ontvangen wij een update.
```

**Expected Trimmed:** `test/trimmed/outlook_dutch_ios.txt`
```
Bedankt voor je bericht.

Met vriendelijke groet

Jan de Vries
Manager Operations
```

**Expected Elided:** `test/elided/outlook_dutch_ios.txt`
```
Verzonden vanuit Outlook voor iOS

Van: Team Support <support@example.com>
Verzonden: dinsdag, september 30, 2025 5:17 PM
Aan: jan.devries@company.nl
Onderwerp: RE: Project aanvraag

Beste Jan,
Graag ontvangen wij een update.
```

---

## Adding New Patterns

### When Tests Fail

When you add a test case and it fails, the gem isn't recognizing some pattern. Here's how to fix it:

#### Step 1: Understand What Failed

The test output shows:
```
Expected (String)                     Actual (String)
Bedankt voor je bericht.              Bedankt voor je bericht.

Met vriendelijke groet                Verzonden vanuit Outlook voor iOS
```

This means the signature line wasn't removed.

#### Step 2: Identify the Pattern Type

Ask yourself:
- Is it a **mobile signature**? → Add to `signature_matcher.rb`
- Is it an **email header** (From:, Sent:, etc.)? → Add to `email_header_matcher.rb`
- Is it a **forwarded message marker**? → Add to `embedded_email_matcher.rb`
- Is it a **delimiter/separator**? → Add to `delimiter_matcher.rb`

#### Step 3: Add the Pattern

**Example: Dutch Outlook iOS signature**

The line is: `Verzonden vanuit Outlook voor iOS`

This is a mobile signature, so edit `lib/email_reply_trimmer/signature_matcher.rb`:

```ruby
SIGNATURE_REGEXES = [
  # ... existing patterns ...

  # Dutch
  /^[[:blank:]]*Verzonden met /i,
  /^[[:blank:]]*Verzonden vanuit /i,  # ← ADD THIS LINE
  /^[[:blank:]]*Verstuurd vanaf mijn /i,
]
```

**Pattern breakdown:**
- `^` - Start of line
- `[[:blank:]]*` - Optional whitespace/tabs at the start
- `Verzonden vanuit ` - The literal text to match
- `/i` - Case-insensitive

#### Step 4: Test Again

```bash
rake test
```

If it passes, you're done! If not, debug further.

#### Step 5: Common Pattern Issues

**Problem: Pattern too specific**
```ruby
# BAD - only matches exact text
/^Verzonden vanuit Outlook voor iOS$/

# GOOD - matches any text after "Verzonden vanuit"
/^[[:blank:]]*Verzonden vanuit /i
```

**Problem: Missing case-insensitive flag**
```ruby
# BAD - won't match "VERZONDEN VANUIT" or "verzonden vanuit"
/^Verzonden vanuit /

# GOOD - matches any case
/^Verzonden vanuit /i
```

**Problem: Not handling whitespace**
```ruby
# BAD - fails if there's a space/tab before the text
/^Verzonden vanuit /i

# GOOD - handles leading whitespace
/^[[:blank:]]*Verzonden vanuit /i
```

---

## Testing Workflow

### Manual Testing

To test a single email without creating test files:

```bash
ruby -e "
require './lib/email_reply_trimmer'

email = File.read('test/emails/your_test.txt')
puts EmailReplyTrimmer.trim(email)
"
```

### Debug Mode - See Pattern Classification

To see how each line is classified:

```ruby
require './lib/email_reply_trimmer'

email = File.read('test/emails/your_test.txt')
EmailReplyTrimmer.preprocess!(email)

email.split("\n").each_with_index do |line, i|
  classification = EmailReplyTrimmer.identify_line_content(line)
  puts "#{i}: [#{classification}] #{line[0..60]}"
end
```

**Output:**
```
0: [t] Thanks for your message.
1: [e]
2: [s] Sent from my iPhone
3: [e]
4: [h] From: John <john@example.com>
5: [h] Sent: Monday, Oct 1, 2025 3:45 PM
```

### Full Test Suite

```bash
# Run all tests
rake test

# Run tests with verbose output
rake test TESTOPTS="-v"

# Run a specific test
ruby -Ilib:test test/test_email_reply_trimmer.rb -n test_outlook_dutch_ios
```

---

## Common Email Patterns

### Outlook Patterns

**English:**
```
From: John Smith <john@example.com>
Sent: Monday, October 1, 2025 3:45 PM
To: Jane Doe <jane@example.com>
Subject: RE: Project Update
```

**Dutch:**
```
Van: Jan de Vries <jan@example.nl>
Verzonden: maandag 1 oktober 2025 15:45
Aan: Piet Jansen <piet@example.nl>
Onderwerp: RE: Project Update
```

**German:**
```
Von: Thomas Müller <thomas@example.de>
Gesendet: Montag, 1. Oktober 2025 15:45
An: Anna Schmidt <anna@example.de>
Betreff: AW: Projekt Update
```

**French:**
```
De : Marie Dubois <marie@example.fr>
Envoyé : lundi 1 octobre 2025 15:45
À : Laurent Martin <laurent@example.fr>
Objet : RE : Mise à jour du projet
```

**Spanish:**
```
De: Carlos García <carlos@example.es>
Enviado: lunes, 1 de octubre de 2025 15:45
Para: Ana Rodriguez <ana@example.es>
Asunto: RE: Actualización del proyecto
```

### Mobile Signatures

**English:**
- `Sent from my iPhone`
- `Sent from Outlook for iOS`
- `Get Outlook for Android`
- `Sent via mobile`

**Dutch:**
- `Verzonden vanuit Outlook voor iOS`
- `Verstuurd vanaf mijn iPhone`
- `Verzonden met BlackBerry`

**German:**
- `Von meinem iPhone gesendet`
- `Von meinem Samsung gesendet`

**French:**
- `Envoyé depuis mon iPhone`
- `Envoyé depuis Yahoo Mail`

### Gmail "On [date] wrote:" Patterns

**English:**
```
On Fri, Oct 3, 2025 at 5:13 PM, John Smith <john@example.com> wrote:
```

**Dutch:**
```
Op 3 okt. 2025 om 17:13 heeft Jan de Vries <jan@example.nl> het volgende geschreven:
```

**German:**
```
Am 3. Okt. 2025 um 17:13 schrieb Hans Müller <hans@example.de>:
```

**French:**
```
Le 3 oct. 2025 à 17:13, Marie Dubois <marie@example.fr> a écrit :
```

---

## Maintenance Instructions for Claude

### When the User Reports a Failing Email

1. **Ask for the full email text** (you'll need to anonymize it)

2. **Create the test case:**
   ```bash
   # Create three files:
   test/emails/descriptive_name.txt       # Full email
   test/trimmed/descriptive_name.txt      # Expected kept content
   test/elided/descriptive_name.txt       # Expected removed content
   ```

3. **Run the test to see what fails:**
   ```bash
   rake test
   ```

4. **Analyze the failure:**
   - Read the test output to see what's being kept vs. what should be kept
   - Use the debug mode to see line classifications
   - Identify which pattern is missing

5. **Add the pattern:**
   - For signatures → `lib/email_reply_trimmer/signature_matcher.rb`
   - For headers → `lib/email_reply_trimmer/email_header_matcher.rb`
   - For "wrote on" → `lib/email_reply_trimmer/embedded_email_matcher.rb`

6. **Test again and iterate:**
   ```bash
   rake test
   ```

7. **Ensure all tests still pass** - your change shouldn't break existing patterns

### Pattern Priority

If you're unsure which matcher to use:
1. Is it mobile/app promotional text? → **Signature Matcher**
2. Is it "From:", "To:", "Sent:", "Subject:"? → **Email Header Matcher**
3. Is it "On [date] wrote:" or "------Original Message------"? → **Embedded Email Matcher**
4. Is it a horizontal line separator? → **Delimiter Matcher**

### Testing Checklist

Before considering the maintenance complete:
- [ ] All existing tests pass (`rake test`)
- [ ] New test case passes
- [ ] Pattern is as general as possible (not too specific to one email)
- [ ] Pattern doesn't accidentally match user content
- [ ] Pattern handles variations (whitespace, case, etc.)

---

## Regular Expressions Quick Reference

### Common Regex Patterns Used

```ruby
^                  # Start of line
$                  # End of line
[[:blank:]]*       # Zero or more spaces/tabs
[[:word:]]+        # One or more word characters (a-z, A-Z, 0-9, _)
.+                 # One or more of any character
.*                 # Zero or more of any character
/i                 # Case-insensitive flag
/m                 # Multiline mode (. matches newlines)
\b                 # Word boundary
\s                 # Whitespace (space, tab, newline)
(?:...)            # Non-capturing group
(?!...)            # Negative lookahead
\d                 # Digit (0-9)
{3,}               # 3 or more repetitions
```

### Testing Regex Patterns

Use [Rubular](https://rubular.com/) to test Ruby regex patterns interactively.

---

## Troubleshooting

### Test Passes But Real Email Fails

**Possible causes:**
1. Character encoding issues (check for hidden characters)
2. Different line endings (CRLF vs LF)
3. HTML in the email (gem only works with plain text)
4. Email preprocessing removed important markers

**Solution:** Get the exact raw email text and create a test case from it.

### Pattern Matches Too Much

**Problem:** Pattern is removing user content that should be kept.

**Solution:** Make the pattern more specific by:
- Adding required surrounding context
- Using word boundaries (`\b`)
- Adding negative lookaheads to exclude certain text

### Pattern Matches Too Little

**Problem:** Pattern isn't catching all variations.

**Solution:** Make the pattern more general by:
- Using `.*` or `.+` instead of specific text
- Adding the `/i` flag for case-insensitivity
- Handling optional whitespace with `[[:blank:]]*`

### Multiple Patterns Conflict

**Problem:** Two matchers classify the same line differently.

**Solution:** Check the order in `identify_line_content()` in `email_reply_trimmer.rb`:
```ruby
def self.identify_line_content(line)
  return EMPTY        if EmptyLineMatcher.match? line
  return DELIMITER    if DelimiterMatcher.match? line
  return SIGNATURE    if SignatureMatcher.match? line
  return EMBEDDED     if EmbeddedEmailMatcher.match? line
  return EMAIL_HEADER if EmailHeaderMatcher.match? line
  return QUOTE        if QuoteMatcher.match? line
  TEXT
end
```

The first match wins. If a signature is being classified as a header, it's because `EmailHeaderMatcher` runs before `SignatureMatcher`.

---

## Recent Changes (October 2025)

### Added Outlook Support

**Problem:** Outlook emails (especially Dutch) weren't being trimmed correctly.

**Solution:** Added patterns for:
1. **Signatures:**
   - `Verzonden vanuit Outlook voor iOS` (Dutch)
   - `Sent from Outlook for iOS` (English)
   - `Get Outlook for iOS/Android` (English promotional)

2. **Email Headers:**
   - `Verzonden:` (Dutch "Sent:")
   - `Envoyé:` (French "Sent:")

3. **Test Cases Added:**
   - `outlook_dutch_ios.txt`
   - `outlook_dutch_signature.txt`
   - `outlook_english_from.txt`
   - `outlook_get_signature.txt`
   - `outlook_german.txt`
   - `outlook_french.txt`
   - `outlook_spanish.txt`

**Files Modified:**
- `lib/email_reply_trimmer/signature_matcher.rb` - Added Outlook signature patterns
- `lib/email_reply_trimmer/email_header_matcher.rb` - Added "Verzonden" and "Envoyé"

**Result:** All 89 tests passing. Outlook emails in 5+ languages now trim correctly.

---

## Summary

This gem uses a sophisticated pattern-matching system to identify and remove quoted email content. The key to maintenance is:

1. **Create good test cases** from real failing emails
2. **Identify which matcher needs updating** (signature, header, embedded, etc.)
3. **Add specific but flexible regex patterns** that handle variations
4. **Test thoroughly** to ensure no regressions

Always work with the test suite - it's your safety net and documentation of expected behavior.
