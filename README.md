# Kimono Backend Project
[commons module](https://github.com/Iotiq-Erste/commons-module.git)

[users module](https://github.com/Iotiq-Erste/users-module.git)

## Running the Project

Run the `Application.java` class in the `application` module.

## Modules

- [commons](https://github.com/inciotiq/commons): Some fundamental classes and methods.
- [user](https://github.com/inciotiq/app-starter-user): Provides user management with roles.

---

## Troubleshooting

- **jar of a module can not be found:** Modules dependent on other modules (like `user`) can not be run on its own. Try
  running the parent project. If `commons` project can't be found, compile the `commons` project and try running the
  parent project. (Running the parent project means running `Application.java` class in the `application` module)
- **Metamodels are missing:** The metamodels are generated when the application starts for the first time. Comment out
  the code that uses metamodels and run the project. Then you will be able to add the metamodels back.