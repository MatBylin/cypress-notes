# 🌲 Cypress – cheatsheet / course notes

A collection of the most important Cypress concepts, commands and patterns. Written in a "get-things-done" style — short, with examples, so I can quickly come back to a topic when I forget how something works.

## 📑 Table of contents

- [Setup and running](#-setup-and-running)
- [Selectors](#-selectors)
- [Lists of elements](#-lists-of-elements)
- [Assertions (should / and / expect)](#-assertions-should--and--expect)
- [Element actions](#-element-actions)
- [Aliases (`.as()` and `@`)](#-aliases-as-and-)
- [`.then()` vs `.should()`](#-then-vs-should)
- [`.invoke()`](#-invoke)
- [Location and navigation](#-location-and-navigation)
- [Hooks](#-hooks)
- [Cypress configuration](#%EF%B8%8F-cypress-configuration)
- [Custom commands](#-custom-commands)
- [Tasks – running code outside the browser](#-tasks--running-code-outside-the-browser)
- [Fixtures](#-fixtures)
- [Stubs (`cy.stub`)](#-stubs-cystub)
- [Intercept – mocking the API](#-intercept--mocking-the-api)
- [API testing – `cy.request`](#-api-testing--cyrequest)
- [Clock and timers](#%EF%B8%8F-clock-and-timers)
- [Tips and best practices](#-tips-and-best-practices)

---

## 🚀 Setup and running

```bash
npm run init                    # init the project (if defined in scripts)
npm install                     # install dependencies
npm install cypress --save-dev  # Cypress itself
npx cypress open                # interactive mode (GUI)
npx cypress run                 # headless mode (CI)
npx cypress run --browser chrome # specific browser
```

> 💡 In `open` mode you can see every step of the test and inspect the DOM at any moment. `run` is much faster — perfect for CI.

---

## 🎯 Selectors

### `cy.get()` vs `cy.find()`

```js
cy.get('a').get('b')   // ❌ WRONG – `b` is searched from the document root
cy.get('a').find('b')  // ✅ RIGHT – `b` is searched inside `a`
```

**Rule:** `get` starts from the document, `find` goes deeper inside the previously found element.

### `data-cy` – the best practice

Instead of relying on CSS classes or IDs, add dedicated `data-cy` attributes in your components:

```js
cy.get("[data-cy='contact-btn-submit']")
```

CSS classes change all the time (style refactors), `data-cy` lives only in tests and nobody will accidentally remove it.

---

## 📋 Lists of elements

```js
cy.get('.task').eq(0)     // element by index (0-based)
cy.get('.task').first()   // first one
cy.get('.task').last()    // last one
cy.get('.task').should('have.length', 3)  // how many there are
```

---

## ✅ Assertions (`should` / `and` / `expect`)

### Most common assertions

```js
cy.get('.toast').should('be.visible')
cy.get('.toast').should('not.exist')          // ⚠️ NOT 'not.be.visible' – `not.exist` means the element is not in the DOM at all
cy.get('input').should('have.value', 'foo')
cy.get('button').should('be.disabled')
cy.get('button').should('have.attr', 'disabled')
cy.get('h1').should('contain.text', 'Welcome')
cy.get('div').should('have.class', 'active')
cy.get('.list').should('have.length', 5)
```

### `should` + `and` (readable chaining)

`and` is just an alias for the next `should` on the same subject — it makes chains more readable:

```js
cy.get('@contactField').parent()
  .should('have.attr', 'class')
  .and('match', /invalid/)
```

---

## 🖱️ Element actions

```js
cy.get('input').type('hello')
cy.get('input').type('{enter}')          // special keys go in curly braces { }
cy.get('input').clear()
cy.get('button').click()
cy.get('input').focus().blur()
cy.get('select').select('Option 1')
cy.get('input[type=checkbox]').check()
```

### `.contains()` returns an element

`contains` is not just an assertion — it returns the element matching the text, so you can chain an action on it:

```js
cy.get("[data-cy='contact-btn-submit']").contains('Send Message').click()
cy.get("[data-cy='contact-btn-submit']").contains('Sending...')
cy.get("[data-cy='contact-btn-submit']").should('have.attr', 'disabled')
```

---

## 🔖 Aliases (`.as()` and `@`)

Instead of querying for the same element multiple times, give it an alias:

```js
cy.get("[data-cy='contact-btn-submit']").as('submitBtn')

cy.get('@submitBtn')
  .contains('Send Message')
  .click()

cy.get('@submitBtn')
  .contains('Sending...')
  .should('have.attr', 'disabled')
```

Aliases also work for intercepts, fixtures and stubs (more on that below).

---

## 🔗 `.then()` vs `.should()`

`.then()` gives you access to the "subject" — the jQuery element, response from a request, etc. — as a regular value:

```js
cy.get("[data-cy='contact-btn-submit']").then((el) => {
  expect(el.text()).to.equal('Send Message')
  el.click()
})
```

**However!** `.should()` with a callback does the same thing but has one massive advantage: **retry**. If an assertion inside fails, Cypress retries until it passes or hits a timeout.

```js
// .then – runs once, if it fails, it fails
cy.get('@contactField').parent().then((el) => {
  expect(el.attr('class')).to.contains('invalid')
})

// .should – will retry until it passes or hits the timeout ✅
cy.get('@contactField').parent().should((el) => {
  expect(el.attr('class')).to.contains('invalid')
})
```

> 🧠 **Rule of thumb:** assertions → `should`, side effects (click, log) → `then`.

---

## 🔧 `.invoke()`

Calls a method or gets a property of the subject:

```js
cy.get('@contactField').parent()
  .invoke('attr', 'class')
  .should('include', 'invalid')

cy.get('input').invoke('val').should('equal', 'foo')
cy.get('.modal').invoke('show')   // calling a jQuery method
```

---

## 🧭 Location and navigation

```js
cy.visit('/')
cy.visit('/about')
cy.location('pathname').should('eq', '/')
cy.url().should('include', '/dashboard')
cy.go('back')
cy.reload()
```

### Screenshot

```js
cy.screenshot()
cy.screenshot('my-name')
```

---

## 🪝 Hooks

```js
before(() => { /* once before all tests in describe */ })
beforeEach(() => { /* before every it */ })
after(() => { /* once after all */ })
afterEach(() => { /* after every it */ })
```

```js
describe('contact form', () => {
  beforeEach('set up test', () => {
    cy.visit('/about')
  })

  it('should send message', () => { /* ... */ })
})
```

### `it.only` – run only this one test

```js
it.only('only this one', () => { /* ... */ })
describe.only('only this block', () => { /* ... */ })
```

Useful while debugging. **Remember to remove it before committing!**

---

## ⚙️ Cypress configuration

`cypress.config.js`:

```js
import { defineConfig } from 'cypress'

export default defineConfig({
  video: false,
  screenshotOnRunFailure: true,
  defaultCommandTimeout: 10000,
  pageLoadTimeout: 10000,
  requestTimeout: 5000,
  e2e: {
    baseUrl: 'http://localhost:5173',
    setupNodeEvents(on, config) {
      // tasks, plugins, event listeners
    },
  },
})
```

### Override inside describe / it

You can override config locally:

```js
describe('contact form', { pageLoadTimeout: 10000, execTimeout: 4000 }, () => {
  it('slow test', { defaultCommandTimeout: 20000 }, () => { /* ... */ })
})
```

---

## 🧱 Custom commands

Reusable "shortcuts". Defined in `cypress/support/commands.js`:

```js
// commands.js
Cypress.Commands.add('login', (email, password) => {
  cy.visit('/login')
  cy.get('[data-cy=email]').type(email)
  cy.get('[data-cy=password]').type(password)
  cy.get('[data-cy=submit]').click()
})
```

Usage in a test:

```js
cy.login('test@example.com', '12345')
```

> 💡 **Queries vs Commands:** Queries (like `get`, `find`, `contains`) are synchronous and chainable. Commands (like `click`, `type`) perform actions. A custom command usually combines both.

---

## 🛠️ Tasks – running code outside the browser

Sometimes you need to run something that doesn't work in the browser — like seeding the database, reading a file from disk, or firing an external request without CORS issues. That's what **tasks** are for.

```js
// cypress.config.js
e2e: {
  setupNodeEvents(on, config) {
    on('task', {
      seedDatabase(filename) {
        // node code, not browser code
        return filename
      },
      async seedDatabaseAsync() {
        await seed()
        return null   // a task MUST return something, even null
      },
    })
  },
},
```

Calling it in a test:

```js
it('should navigate between pages', () => {
  cy.task('seedDatabase', 'filename.sql').then((returnValue) => {
    console.log(returnValue)
  })
})

beforeEach(() => {
  cy.task('seedDatabase')
  cy.visit('/')
})
```

---

## 📦 Fixtures

Static test data kept in `cypress/fixtures/`:

```js
cy.fixture('user-location.json').as('user-location')

cy.get('@user-location').then(fakePosition => {
  console.log(fakePosition.coords.latitude)
})
```

The most common use case: returning fixture data as a response inside `cy.intercept()`.

---

## 🤖 Stubs (`cy.stub`)

Stubs replace the implementation of some function in the browser. Perfect for APIs we don't control, e.g. geolocation.

> 🔑 **Rule:** `cy.intercept()` for mocking the API, `cy.stub()` for mocking browser behavior.

```js
describe('share location', { defaultCommandTimeout: 10000 }, () => {
  beforeEach('set up test', () => {
    cy.visit('/').then((window) => {
      cy.stub(window.navigator.geolocation, 'getCurrentPosition')
        .as('position')
        .callsFake((callBack) => {
          callBack({
            coords: {
              latitude: 37.5,
              longitude: 47.02,
            },
          })
        })
    })
  })

  it('should pass fake location to the app', () => {
    cy.get('[data-cy=share-btn]').click()
    cy.get('@position').should('have.been.called')
  })
})
```

---

## 🌐 Intercept – mocking the API

`cy.intercept()` can be used in two ways:

### 1. As a spy (without overriding the response)

```js
cy.intercept('POST', '/newsletter*').as('subscribe')
// the request still hits the real endpoint, but we can inspect it
```

### 2. As a full mock (with a stubbed response)

```js
it('should sign to the newsletter', () => {
  cy.intercept('POST', '/newsletter*', { statusCode: 201 }).as('subscribe')

  cy.get("[data-cy='newsletter-email']").type('example@wp.pl')
  cy.get("[data-cy='newsletter-submit']").click()
  cy.wait('@subscribe')                       // ⏳ waits until the request fires
  cy.contains('Thanks for signing up')
})
```

### Using a fixture as the body

```js
cy.intercept('GET', '/api/users', { fixture: 'users.json' }).as('getUsers')
```

### Asserting on the sent request

```js
cy.wait('@subscribe').its('request.body').should('deep.equal', {
  email: 'example@wp.pl',
})
```

---

## 🔌 API testing – `cy.request`

`cy.request` hits the API directly, bypassing the browser. Great for backend tests or for setting up state before a UI test.

```js
it('should successfully create a new contact', () => {
  cy.request({
    method: 'POST',
    url: '/newsletter',
    body: { email: 'test@example.com' },
    form: true,
  }).then((response) => {
    expect(response.status).to.eq(201)
    expect(response.body).to.have.property('id')
  })
})
```

> 💡 If `baseUrl` is set in the config, you can pass just the path (`/newsletter`).

---

## ⏱️ Clock and timers

Sometimes you want to "skip ahead" instead of really waiting 2s for a `setTimeout`:

```js
cy.clock()         // takes control of the clock
cy.tick(2000)      // moves time forward by 2 seconds
```

Awesome for testing throttle / debounce / animations without slowing the test suite down.

---

## 🧙 Tips and best practices

### ❌ What to avoid

- **`cy.wait(2000)` with a number** – it's always either too short (flaky) or too long (slow tests). Use `cy.wait('@alias')` for intercepts; assertions retry on their own.
- **Selectors based on CSS classes / inner text** – use `data-cy` instead.
- **Sharing state between tests** – every `it` should be independent. Set up state in `beforeEach`.
- **Logging in via the UI in every test** – use `cy.session()` or `cy.request()` for programmatic login.

### ✅ Cypress waits for you

Commands have built-in retry / timeout (`defaultCommandTimeout`). You don't need to write:

```js
// ❌ don't do this
cy.wait(2000)
cy.get('.toast').should('be.visible')

// ✅ this is enough
cy.get('.toast').should('be.visible')   // it'll wait up to 4-10s anyway
```

### Test structure (AAA)

```js
it('should X', () => {
  // Arrange – setup
  cy.task('seedDatabase')
  cy.visit('/')

  // Act – action
  cy.get('[data-cy=submit]').click()

  // Assert – check
  cy.contains('Success').should('be.visible')
})
```

### Environment variables

In `cypress.config.js`:

```js
env: { apiUrl: 'http://localhost:3000' }
```

In a test:

```js
cy.request(`${Cypress.env('apiUrl')}/users`)
```

---

## 📚 Useful links

- [Cypress documentation](https://docs.cypress.io)
- [List of all assertions](https://docs.cypress.io/guides/references/assertions)
- [Best practices](https://docs.cypress.io/guides/references/best-practices)
- [Real World App – sample project with tests](https://github.com/cypress-io/cypress-realworld-app)

---

