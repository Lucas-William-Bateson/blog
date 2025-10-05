---
title: "Ingest: Building a Nutrition Logger with React Native, OpenAI, and Apple Health"
description: "Lust to learn -> fully functioning app in one evening: how I built a simple nutrition logging app using React Native, OpenAI Vision, and Apple HealthKit."
pubDate: 2025-10-03
updatedDate: 2025-10-05
tags: ["React Native", "AI", "Apple Health", "iOS"]
draft: false
---

For a while, I’d been looking to learn how to make apps. All I really wanted was to build something dead simple, but still to let me understand how apps work. So I built one myself.

---

## Starting with the Stack

I could have gone full SwiftUI, but React Native made more sense. I know React inside-out, and with libraries like `react-native-health` I could still access Apple HealthKit without writing native Swift code manually. The goal was fast iteration and complete control.

The core stack ended up being:

- **React Native** — for UI and logic
- **react-native-image-picker** — to capture food photos
- **OpenAI Vision (Responses API)** — to analyze images and estimate nutrients
- **react-native-health** — to write all nutrients into Apple HealthKit

This combination gave me the best of both worlds: a flexible JS-based UI and full native device integration.

---

## Bootstrapping the App

I started by generating a new React Native project:

```bash
npx react-native init Ingest
cd Ingest
```

Then I installed the key dependencies:

```bash
npm install react-native-image-picker react-native-health react-native-permissions
cd ios && pod install && cd ..
```

On the iOS side, I opened the project in Xcode, enabled **HealthKit** under _Signing & Capabilities_, and added the required privacy strings to `Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>Take food photos for nutrition logging</string>
<key>NSHealthShareUsageDescription</key>
<string>Read and write nutrition data to Apple Health</string>
<key>NSHealthUpdateUsageDescription</key>
<string>Write nutrients to Apple Health</string>
```

Without these, iOS either rejects the build or silently denies access to Health.

---

## Wiring Up HealthKit

The `react-native-health` library exposes a simple API for both permissions and data writes. My first step was to request write permissions for all the nutrients I cared about:

```javascript
import AppleHealthKit, { HealthKitPermissions } from "react-native-health";

const permissions = {
  permissions: {
    read: [],
    write: [
      AppleHealthKit.Constants.Permissions.DietaryEnergyConsumed,
      AppleHealthKit.Constants.Permissions.DietaryProtein,
      AppleHealthKit.Constants.Permissions.DietaryCarbohydrates,
      AppleHealthKit.Constants.Permissions.DietaryFatTotal,
      AppleHealthKit.Constants.Permissions.DietaryFiber,
      AppleHealthKit.Constants.Permissions.DietarySugar,
      AppleHealthKit.Constants.Permissions.DietarySodium,
      AppleHealthKit.Constants.Permissions.DietaryPotassium,
      AppleHealthKit.Constants.Permissions.DietaryVitaminC,
      // ...and more as needed
    ],
  },
};

AppleHealthKit.initHealthKit(permissions, (err) => {
  if (err) {
    console.error("HealthKit error", err);
    return;
  }
  console.log("HealthKit ready");
});
```

Once permissions were granted, I could call `AppleHealthKit.saveFood(...)` with energy, macros, and dates. This gave me instant feedback in Apple Health after each “meal”.

---

## Capturing Food Photos

Next up: the camera. With `react-native-image-picker`, this was almost too easy:

```javascript
import { launchCamera } from "react-native-image-picker";

launchCamera({ mediaType: "photo" }, (response) => {
  if (response.assets && response.assets[0]) {
    setPhoto(response.assets[0]);
  }
});
```

The result is a local URI I can convert to base64 and send to OpenAI.

---

## Using OpenAI Vision for Nutrient Estimation

This is where the magic happens. Instead of me manually entering “200g chicken breast + 1 cup rice”, I send the **photo** and a **short description** (like “1/4 of this”) to OpenAI’s Responses API using a JSON schema.

```javascript
const body = {
  model: "gpt-4.1",
  input: [
    {
      role: "system",
      content:
        "You are a nutrition estimation model. Return JSON with nutrients for the portion.",
    },
    {
      role: "user",
      content: [
        { type: "input_text", text: portionText },
        { type: "input_image", image_url: base64Image },
      ],
    },
  ],
  response_format: { type: "json_object" },
};
```

The model returns a neat JSON object:

```json
{
  "energy_kcal": 430,
  "protein_g": 26,
  "carbs_g": 32,
  "fat_g": 18,
  "fiber_g": 4,
  "sodium_mg": 500,
  "vitamin_c_mg": 12
}
```

I then feed this straight into `AppleHealthKit.saveFood`. No UI lists, no manual lookups. It just works.

---

## Logging to Apple Health

Once the JSON arrives, saving is a one-liner:

```javascript
AppleHealthKit.saveFood(
  {
    foodName: "Meal",
    energyConsumed: data.energy_kcal,
    protein: data.protein_g,
    carbohydrates: data.carbs_g,
    fatTotal: data.fat_g,
    date: new Date().toISOString(),
  },
  (err) => {
    if (err) console.error("Save error:", err);
  }
);
```

Opening the Health app afterward and seeing everything logged automatically is deeply satisfying.

---

## Final Result: Ingest

The finished app does exactly what I needed: open it, snap a photo of your meal, type a quick description like "half the plate" or "one bowl," and hit send. Within seconds, calories, macros, and micronutrients appear in Apple Health, ready to integrate with the rest of your health data. No searching through food databases, no wrestling with portion size sliders, no multi-step logging flows.

It's just a thin React Native shell—no database, no backend, no complex state management. The phone's camera feeds OpenAI, OpenAI feeds HealthKit, and you get on with your day.

---

## Full Source Code

After writing this section I have made some changes to the structure of libraries used due to technical limitations, but the core idea remains the same.

You can find the full code on my GitHub: 👉 [Ingest](https://github.com/Lucas8448/ingest)

---
