---
name: java-code-formatter
description: "Auto-format Java source code after generating or modifying files. Run ./gradlew spotlessApply to enforce Google Java Style via the Eclipse JDT formatter configured in spotless.properties."
---

# Java Code Formatter

## When to Activate

After **any** skill or task that creates or modifies `.java` source files — including but not limited to unit tests, Spring Boot classes, JPA entities, Kafka consumers/producers, Temporal workflows/activities, REST controllers, and utility classes.

## Rule

After writing or editing Java files, **always** run:

```bash
./gradlew spotlessApply
```

This formats all Java sources using **Google Java Style** via the Eclipse JDT formatter configured in `spotless.properties` at the project root.

## Notes

- Run this **once** at the end of a batch of changes, not after every individual file edit.
- If `spotlessApply` reports errors, fix them before proceeding.
- Do **not** manually format code to match the style — let Spotless handle it.
