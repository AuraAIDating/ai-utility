---
name: healthcare-domain
description: Healthcare compliance patterns for HHA MVP. Covers HIPAA PHI safeguards, NPI Luhn validation, PECOS status, ICD-10/HCPCS code validation with Levenshtein distance correction (POL-126), Medicaid/Title XIX requirements, X12 EDI 270/271/278 structure via staedi, SWO/ABN/AOB document requirements, and PiiMaskingUtil patterns.
---

# Healthcare Domain Skill

## 2. NPI Validation (Luhn Algorithm)

National Provider Identifier (NPI) is a 10-digit number. The 10th digit is a Luhn check digit.

```java
public final class NpiValidator {

    public static boolean isValid(String npi) {
        if (npi == null || !npi.matches("\\d{10}")) return false;

        // Prepend "80840" as per NPI Luhn specification
        String fullNpi = "80840" + npi;
        return luhnCheck(fullNpi);
    }

    private static boolean luhnCheck(String number) {
        int sum = 0;
        boolean alternate = false;
        for (int i = number.length() - 1; i >= 0; i--) {
            int n = number.charAt(i) - '0';
            if (alternate) {
                n *= 2;
                if (n > 9) n -= 9;
            }
            sum += n;
            alternate = !alternate;
        }
        return sum % 10 == 0;
    }

    private NpiValidator() {}
}
```

---

## 3. HCPCS and ICD-10 Validation (POL-126)

### HCPCS Level II Format

```java
public final class HcpcsValidator {
    // Format: one letter followed by 4 digits — e.g., A6197, E0185
    private static final Pattern HCPCS_PATTERN = Pattern.compile("[A-Z][0-9]{4}");

    public static boolean isValidFormat(String code) {
        return code != null && HCPCS_PATTERN.matcher(code).matches();
    }
}
```

### ICD-10 Format

```java
public final class Icd10Validator {
    // Format: letter + 2 digits + optional decimal + 1-4 alphanumeric
    private static final Pattern ICD10_PATTERN =
        Pattern.compile("[A-Z][0-9]{2}(\\.[A-Z0-9]{1,4})?");

    public static boolean isValidFormat(String code) {
        return code != null && ICD10_PATTERN.matcher(code).matches();
    }
}
```

### Levenshtein Distance Code Correction (POL-126)

When OCR extracts a code that doesn't match the reference table, use Levenshtein to suggest corrections.

```java
public final class LevenshteinDistance {

    public static int compute(String s1, String s2) {
        int m = s1.length(), n = s2.length();
        int[][] dp = new int[m + 1][n + 1];
        for (int i = 0; i <= m; i++) dp[i][0] = i;
        for (int j = 0; j <= n; j++) dp[0][j] = j;
        for (int i = 1; i <= m; i++) {
            for (int j = 1; j <= n; j++) {
                if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                    dp[i][j] = dp[i - 1][j - 1];
                } else {
                    dp[i][j] = 1 + Math.min(dp[i - 1][j - 1],
                                   Math.min(dp[i - 1][j], dp[i][j - 1]));
                }
            }
        }
        return dp[m][n];
    }

    // Find top-N closest codes from a list of valid codes
    public static List<String> topCorrections(String invalidCode,
                                               List<String> validCodes,
                                               int topN) {
        return validCodes.stream()
            .filter(code -> LevenshteinDistance.compute(invalidCode, code) <= 2)
            .sorted(Comparator.comparingInt(code ->
                LevenshteinDistance.compute(invalidCode, code)))
            .limit(topN)
            .collect(Collectors.toList());
    }
}
```

**Usage in activity (POL-126):**

```java
// 1. Check exact match in HCPCS_CODE table
Optional<HcpcsCode> exact = hcpcsCodeRepository.findById(extractedCode);
if (exact.isEmpty()) {
    // 2. Try Levenshtein correction
    List<String> allCodes = hcpcsCodeRepository.findAllCodes();
    List<String> suggestions = LevenshteinDistance.topCorrections(extractedCode, allCodes, 3);
    if (!suggestions.isEmpty()) {
        String correctedCode = suggestions.get(0);
        log.info("hcpcs_correction original={} corrected={} orderId={}",
            extractedCode, correctedCode, orderId);
        orderProductRepository.updateCorrectedHcpcsCode(productId, correctedCode);
    }
}
```

## 6. HIPAA Minimum Necessary for AI Calls

**When calling Gemini or ML model — NEVER include:**

- Patient SSN
- Full DOB (send age range or year only if needed)
- Full patient name
- Insurance policy numbers

**Safe to send to Gemini/ML:**

- ICD-10 codes
- HCPCS codes
- Wound measurements (without patient identity)
- Document OCR text (after de-identification)
- HCPCS + payer ID combination (for ML model)

---

## 7. Wound Assessment Completeness Rules

For wound-related orders (ICD-10 codes in L89.x range — pressure injuries):

```java
void validateWoundData(WoundAssessment wound) {
    if (wound.stage() == null) throw new InvalidOrderException("Wound stage required");
    if (wound.location() == null) throw new InvalidOrderException("Wound location required");
    if (wound.lengthCm() == null || wound.widthCm() == null || wound.depthCm() == null) {
        throw new InvalidOrderException("Wound dimensions (L x W x D) required");
    }
    if (wound.tissueType() == null) throw new InvalidOrderException("Tissue type required");
    // ICD-10 stage must match wound stage
    if (!icd10MatchesWoundStage(wound.diagnosisCode(), wound.stage())) {
        throw new InvalidOrderException("ICD-10 code stage mismatch with wound stage");
    }
}
```
