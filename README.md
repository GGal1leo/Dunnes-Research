# Static Analysis of the Dunnes Stores ValueClub Android Application: Uncovering the Voucher and Authentication Mechanisms

**Author:** Daniel Vetrila
**Date:** September 14, 2025
**Version:** 1.0

## Abstract

This paper details the static analysis of the `com.dunnes.valueclubcard` Android application. The primary objective was to understand the underlying architecture and API communication protocols related to its core features: user authentication and the management of loyalty vouchers and gift cards. By decompiling the application's APK and systematically tracing the code flow from UI components to the networking layer, we successfully mapped the application's architecture, identified all relevant API endpoints, and reconstructed the precise OAuth 2.0 authentication request. This research serves as a case study in modern mobile application reverse engineering for security and architectural analysis.

---

## 1. Introduction

The Dunnes Stores ValueClub application is the digital portal for a major retail loyalty program. It allows users to manage their account, view and redeem digital vouchers, track loyalty points, and manage gift cards. The goal of this research was to reverse engineer the application's client-side logic to understand how this data is securely obtained and managed.

This investigation was conducted purely through **static analysis**, meaning the application package (APK) was examined without running the application or intercepting live network traffic. This approach allows for a deep understanding of the app's intended logic and architecture as designed by its developers.

## 2. Methodology

The primary tool used for this analysis was the **JADX decompiler**, which converts Android's DEX bytecode into largely readable Java/Kotlin code. The methodology followed a logical "top-down" tracing approach:

1.  **Resource Mapping:** Began by analyzing the auto-generated `R.java` file to identify key UI layouts, strings, and IDs related to the target features.
2.  **Code Tracing:** Used the resource IDs as entry points to locate the corresponding `Activity` or `Fragment` classes controlling the user interface.
3.  **Dependency Following:** Systematically traced the flow of logic from the UI layer through various architectural layers (ViewModels, UseCases, Repositories) to pinpoint the networking code.
4.  **Endpoint Reconstruction:** Analyzed the networking code, specifically Retrofit interfaces, to identify and document the full API endpoints and their required parameters.

## 3. Architectural Analysis & Findings

The application employs a modern, layered architecture, indicative of professional software design patterns. This separation of concerns made the tracing process logical and predictable.

### 3.1. Initial Resource Mapping (`R.java`)

The `R.java` file provided the initial breadcrumbs. Keywords such as `voucher`, `rewardVoucher`, `giftCard`, and `valueClubCard` were found in layout (`R.layout`), string (`R.string`), and ID (`R.id`) definitions. These references pointed directly to the UI components responsible for displaying voucher information.

### 3.2. Background Sync Mechanism (`VouchersBackgroundWorker`)

Tracing the usage of voucher-related layouts led to the discovery of `eu.buymie.customer.background.VouchersBackgroundWorker`. This class, an Android `CoroutineWorker`, is responsible for periodically fetching voucher data in the background.

This class provided the first major breakthrough by revealing an explicitly constructed URL for fetching active gift cards:

```java
// Inside VouchersBackgroundWorker.java
String f6 = AbstractC0105m.f(AbstractC0267y.d("https://", c.f2813g), "v7/app/GiftCard/Active");
```

This line confirmed the endpoint `/v7/app/GiftCard/Active` and pointed to a utility class (`C8.c`) for the base URL.

### 3.3. Uncovering the UseCase-Repository Pattern

The `VouchersBackgroundWorker` did not contain networking code itself. Instead, it depended on a class obfuscated as `Uc.C1070j1`, which was aliased as `dunnesUseCase`. This `UseCase` class, in turn, depended on a `Repository` class, `Oc.C0935k0`.

This revealed a classic **UseCase-Repository** architecture:
-   **Worker:** Schedules the background task.
-   **UseCase (`Uc.C1070j1`):** Defines *what* business logic to execute (e.g., "get vouchers").
-   **Repository (`Oc.C0935k0`):** Knows *how* to get the data (from a network source).

### 3.4. The Networking Layer: Retrofit Service Interface

The `Repository` class (`Oc.C0935k0`) made calls using a Retrofit service interface, `Nc.l`. Decompiling this interface revealed a clear, declarative list of the application's primary data-fetching endpoints.

### 3.5. Barcode Decoding (`cb.c`)

A separate analysis of a class named `cb.c` revealed it to be a decoder for **GS1 DataBar Expanded** barcodes, utilizing the ZXing library. This component is not used for obtaining voucher data from the API but is crucial for the in-store redemption process, where a cashier's scanner reads the barcode displayed on the app screen.

## 4. Authentication Flow Analysis

A separate investigation was required to understand the authentication mechanism.

### 4.1. Locating the Login Logic

Using `R.id.action_activity_sign_in_login` from the resource file, we located `eu.buymie.customer.views.activities.dunnes.dunnesSignIn.DunnesSignInActivity`. The `onClick` listener for the login button initiated a call to a `ViewModel`.

### 4.2. Identifying the Authentication Endpoint

The `SignInActivity` calls a `ViewModel` (`de.d`), which contains the following code:
```java
// Inside de/d.java (ViewModel)
String f6 = AbstractC0105m.f(AbstractC0267y.d("https://", C8.c.f2812f), "connect/token");
```
This revealed the authentication endpoint path: `/connect/token`.

Further investigation of the utility class `C8.c` revealed the base URLs:
```java
// Inside C8/c.java
public static String f2812f = "demomy.buymie.eu"; // Used for login in this build
public static String f2813g = "api.buymie.eu/";   // Used for main API calls
```This indicated that the authentication server is distinct from the main API server. For the production application, the login endpoint resolves to **`https://my.buymie.eu/connect/token`**.

### 4.3. Determining the Authentication Protocol

The final piece of the puzzle was found in the task object `Uc.H1`, which is executed by the `ViewModel`. This class contained the method call that constructs the network request:
```java
// Inside Uc/H1.java
obj = q02.h(this.c, this.f2601d, this.f2602e, this.f2603f, "password", this);
```
The hardcoded `"password"` string is the definitive clue, identifying the authentication scheme as the **OAuth 2.0 Resource Owner Password Credentials Grant**. This protocol requires the credentials to be sent as `application/x-www-form-urlencoded` data.

### 4.4. The `client_id`

The final required parameter, `client_id`, was found as a string resource referenced in `R.java` (`R.string.client_id`) and defined in `res/values/strings.xml`.

## 5. Summary of Identified Endpoints

### 5.1. Authentication API

| HTTP Method | Base URL | Endpoint Path | Content-Type | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `https://my.buymie.eu` | `/connect/token` | `application/x-www-form-urlencoded` | Obtains a bearer token using user credentials. |

### 5.2. Application Data API

*All endpoints require an `Authorization: Bearer <token>` header.*

| HTTP Method | Base URL | Endpoint Path | Purpose |
| :--- | :--- | :--- | :--- |
| `GET` | `https://api.buymie.eu` | `/v7/voucher/GetActiveV2` | Fetches active vouchers. |
| `GET` | `https://api.buymie.eu` | `/v7/voucher/GetHistory` | Fetches redeemed/expired vouchers. |
| `GET` | `https://api.buymie.eu` | `/v7/voucher/LastStatement` | Fetches the latest points statement. |
| `GET` | `https://api.buymie.eu` | `/v7/ValueClub/Points` | Fetches the current loyalty points balance. |
| `GET` | `https://api.buymie.eu` | `/v7/ValueClub/Summary` | Fetches a loyalty account summary. |
| `GET` | `https://api.buymie.eu` | `/v7/app/GiftCard/Active` | Fetches active gift cards. |

## 6. Conclusion

The static analysis of the `com.dunnes.valueclubcard` application was highly successful. It revealed a modern, well-structured application that separates concerns into distinct architectural layers. Vouchers and loyalty data are managed entirely on the server-side and are retrieved via a standard RESTful API. Authentication is handled by a dedicated identity server using the OAuth 2.0 protocol. By systematically tracing the code from the user interface down to the networking layer, it was possible to fully reconstruct the application's API communication model.

## 7. Disclaimer

This research was conducted for security analysis purposes only. The information contained herein is intended to provide insight into mobile application architecture and security. Any attempt to use this information to access data without authorization or for fraudulent purposes is illegal and unethical.
