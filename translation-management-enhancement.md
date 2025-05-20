# Proposal: Enhancing Translation Management

## 1. Current Situation

The project currently utilizes static JSON files (`en.json`, `ar.json`, etc.) located within the `packages/locales/src/` directory. These translations are loaded dynamically on the server-side via a `getDictionary` function and consumed in client components using a React Context (`TranslationsContext`) and a custom hook (`useTranslations`).

While functional, this setup requires developer intervention and a full application redeployment to update any translations.

## 2. Objective

To implement a system that allows administrators to manage and update application translations easily and securely, without requiring frequent developer involvement or application redeployments. This solution needs to be mindful of a tight project deadline, prioritizing a balance between functionality and development effort.

## 3. Explored Options (Summary)

Several approaches were considered:

- **Full Database Solution:** Storing translations in a database with a comprehensive admin UI.
  - _Pros:_ Granular control.
  - _Cons:_ High development workload for schema, backend, and UI.
- **Headless CMS:** Using a third-party CMS for translation management.
  - _Pros:_ Offloads UI and storage.
  - _Cons:_ External dependency, integration effort.
- **Git-based Workflow:** Admin UI commits changes to JSON files in Git, triggering redeploys.
  - _Pros:_ Leverages existing Git.
  - _Cons:_ Complex UI, still requires redeployments.

## 4. Recommended Solution: External JSON File Storage (AWS S3) with Admin Interface

This approach is recommended for its balance of flexibility, security, and manageable development effort within a constrained timeframe.

### 4.1. Core Concept

- **External Storage:** Translation JSON files (e.g., `en.json`, `ar.json`) will be stored in an AWS S3 bucket.
- **Dynamic Fetching:** The `getDictionary` function in `packages/locales/src/index.ts` will be modified to fetch these JSON files from S3 via HTTP(S) instead of importing them directly from the local filesystem.
- **Admin Management:** A dedicated section in the `3apaq-admin` application will allow authenticated administrators to manage these translations.

### 4.2. Implementation Details & User Preferences

The preferred implementation details are:

- **Primary Admin Interface (Preferred):**

  - Develop a page within the `3apaq-admin` application that allows administrators to directly view and edit individual key-value pairs for each locale.
  - Upon saving, the admin application will update the corresponding JSON file in the S3 bucket.
  - This provides a user-friendly experience for admins.

- **Alternative/Interim Admin Interface (To Minimize Initial Development):**

  - As a faster-to-implement alternative, or as an initial version, the admin page could allow:
    - Downloading the current JSON translation file (e.g., `en.json`) from S3.
    - Admins can then edit this file using external software (like a text editor or specialized JSON editor).
    - Re-uploading the modified JSON file to S3, overwriting the previous version.
  - This significantly reduces the development time for the admin UI while still enabling admin-driven updates.

- **Initial Seeding & Fallback Mechanism:**

  - The application will be initially seeded with completed `en.json` and `ar.json` files. These files will reside locally within the `packages/locales/src/` directory (or a designated fallback location).
  - These local files will also be uploaded to the S3 bucket as the initial source of truth for dynamic fetching.
  - **Fallback:** If the application fails to fetch a translation file from S3 (e.g., network error, S3 issue, corrupted file), it will fall back to using these locally stored, complete, and verified `en.json` and `ar.json` files. These local fallback files should ideally remain unchanged by the admin process unless a deliberate synchronization step is implemented.
  - **Synchronization (Optional Consideration):** A future enhancement could involve a mechanism to periodically sync the S3 versions back to the local fallback files, or a manual process for developers to update them.

- **Caching:**

  - To optimize performance and reduce S3 costs, responses from S3 (the fetched JSON translation files) will be cached on the application server.
  - The `getDictionary` function will first check the cache before attempting to fetch from S3. A suitable cache duration (e.g., 5-15 minutes, or configurable) will be implemented to ensure timely updates.

- **Security:**
  - The S3 bucket will be configured with appropriate permissions.
  - The `3apaq-admin` application will use secure AWS credentials (e.g., an IAM role or dedicated IAM user with least-privilege access) to write/update files in the S3 bucket.
  - The main application (fetching translations) will use credentials with read-only access to the S3 bucket.

### 4.3. Example: Modified `getDictionary` (Conceptual)

```typescript
// packages/locales/src/index.ts (Conceptual - to be adapted)
import "server-only";
import localEnTranslations from "./en.json"; // Local fallback
import localArTranslations from "./ar.json"; // Local fallback

export type Dictionary = {
  [K in keyof typeof localEnTranslations]: string;
};

// In-memory cache (example, could be more sophisticated e.g. Redis)
const translationCache = new Map<
  string,
  { data: Dictionary; timestamp: number }
>();
const CACHE_DURATION_MS = 5 * 60 * 1000; // 5 minutes

const fetchDictionaryFromS3 = async (
  locale: string
): Promise<Dictionary | null> => {
  try {
    const response = await fetch(
      `${process.env.TRANSLATION_STORAGE_BASE_URL}/${locale}.json`
    );
    if (!response.ok) {
      console.error(
        `Failed to fetch ${locale}.json from S3: ${response.status}`
      );
      return null;
    }
    return (await response.json()) as Dictionary;
  } catch (error) {
    console.error(`Error fetching ${locale}.json from S3:`, error);
    return null;
  }
};

const getLocalFallback = (locale: string): Dictionary => {
  if (locale === "ar") return localArTranslations as Dictionary;
  return localEnTranslations as Dictionary; // Default to English
};

const dictionaries: Record<string, () => Promise<Dictionary>> = {
  en: async () => {
    const cached = translationCache.get("en");
    if (cached && Date.now() - cached.timestamp < CACHE_DURATION_MS) {
      return cached.data;
    }
    const s3Data = await fetchDictionaryFromS3("en");
    if (s3Data) {
      translationCache.set("en", { data: s3Data, timestamp: Date.now() });
      return s3Data;
    }
    return getLocalFallback("en"); // Fallback to local
  },
  ar: async () => {
    const cached = translationCache.get("ar");
    if (cached && Date.now() - cached.timestamp < CACHE_DURATION_MS) {
      return cached.data;
    }
    const s3Data = await fetchDictionaryFromS3("ar");
    if (s3Data) {
      translationCache.set("ar", { data: s3Data, timestamp: Date.now() });
      return s3Data;
    }
    return getLocalFallback("ar"); // Fallback to local
  },
  // ... other locales can be added similarly
};

export const getDictionary = async (locale: string): Promise<Dictionary> => {
  const loadDictionary = dictionaries[locale] || dictionaries.en; // Default to 'en' if locale specific loader not found
  if (!loadDictionary) {
    // This case should ideally not be reached if 'en' is always present
    console.error(
      `Dictionary loader for locale "${locale}" not found, and 'en' default failed.`
    );
    return getLocalFallback("en"); // Absolute fallback
  }
  try {
    return await loadDictionary();
  } catch (error) {
    console.error(
      `Critical error loading dictionary for locale "${locale}":`,
      error
    );
    return getLocalFallback(locale); // Fallback to local for the requested/default locale
  }
};
```

### 4.4. Advantages

- **No Redeployment for Updates:** Admins can change translations, and they reflect after the cache expires, without needing a new build/deploy.
- **Secure:** Utilizes AWS S3's robust security and IAM for access control.
- **Manages Development Workload:**
  - The "download/edit/upload" admin flow is quick to implement.
  - The key-value editor can be a subsequent enhancement if time permits.
- **Familiar Format:** Continues using JSON, which developers and potentially admins are used to.
- **Scalability & Reliability:** AWS S3 is highly scalable and reliable.
- **Robust Fallback:** Ensures the application remains functional even if S3 is temporarily unavailable or a file is corrupted.

## 5. Next Steps & Action Items

1.  **Setup AWS S3 Bucket:**
    - Create an S3 bucket.
    - Configure CORS policies if direct browser uploads are considered for the admin UI.
    - Set up IAM roles/users for read access (application) and write access (admin app).
2.  **Upload Initial Translations:** Upload the current `en.json` and `ar.json` to the S3 bucket.
3.  **Modify `getDictionary`:**
    - Implement fetching logic from S3.
    - Implement caching.
    - Implement the fallback to local files.
    - Ensure `TRANSLATION_STORAGE_BASE_URL` is configured via environment variables.
4.  **Develop Admin Interface (`3apaq-admin`):**
    - **Phase 1 (MVP):** Implement the download/upload functionality for JSON files.
    - **Phase 2 (Enhancement):** Develop the key-value pair editing interface.
5.  **Testing:** Thoroughly test the new translation fetching, caching, fallback, and admin update workflows.
6.  **Documentation:** Update any relevant developer or admin documentation.

This approach provides a clear path to achieving the desired translation management capabilities while respecting the project's timeline.
