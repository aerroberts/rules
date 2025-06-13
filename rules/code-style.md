# Typescript Code Style

When we are writing typescript projects, its important that we follow a clear structure and style guidance.
Adhearance to this gives us better quality code in the long run. Since much of the code will be AI generated, sticking to these rules will help us maintain the codebase.

The overarching principles we want to be aware of are:
1. Class-first approach, without many (or any) global variables or global singletons. These patterns are hard to test and maintain.
2. Interfaces are king. As systems get bigger, dedicated interfaces are the best way to define contracts between components. Use them sparingly but are pivitol in critial sections.
3. Errors should never be generic. Throw them often, and make them specific. Custom error types allows error handling to be more effective and less flakey.
4. Tests as specifications. Tests define how you want things to function, what behaviour you expect, and are the effective set of feature requirements of the system.

## Class first approach

We want to always jump for classes for constructing the system. This is critical to unlock solid testing.
Classes should be conceptually structured like below:

```ts

import { Castle } from '../castles/castle';

// Errors are defined in a separate file and are typed as a custom error type
import { DragonError } from './errors';

// Types are defined in a separate file and are used to type the parameters of the class
// We want to avoid interfaces within classes and instead use a types file;
import type { DragonParams } from './types';

class Dragon {

    private dragonName: string;
    private residence: Castle;
    private isDead: boolean;
    private age: number;

    constructor (params: DragonParams) {
        this.dragonName = params.name;
        this.residence = params.residence;
        this.isDead = params.isDead ?? false;
        this.age = params.age ?? 0;
    }


    // Private Utilities

    private _properName () {
        return `${this.dragonName} of ${this.residence.getName()}`
    }

    private _areOtherDragons () {
        return this.residence.getDragons().length > 1;
    }

    // Public API

    /**
     * Changes the dragon's residence to a new castle
     * @param newResidence - The castle to move to
     * @throws DragonError if the new residence is the same as current
     */
    public changeResidence(newResidence: Castle): void {
        if (newResidence === this.residence) {
            throw new DragonError('Cannot move to same residence');
        }
        this.residence = newResidence;
    }

    /**
     * Creates a new baby dragon with the given name
     * @param babyName - Name for the new dragon
     * @returns The newly created baby dragon
     * @throws DragonError if the dragon is too young to have babies
     */
    public haveBaby(babyName: string): Dragon {
        if (this.age < 100) {
            throw new DragonError('Dragon must be at least 100 years old to have babies');
        }
        return new Dragon(babyName, this.residence);
    }

    /**
     * Kills the dragon, removing it from its residence
     * @throws DragonError if the dragon is already dead
     */
    public kill(): void {
        if (this.isDead) {
            throw new DragonError('Dragon is already dead');
        }
        this.isDead = true;
        this.residence.removeDragon(this);
    }

}

```


## Interfaces are king

Interfaces allow us to reason about the system more abstractly, then enforce that system is correct. 
Interfaces in typescript are a specific type definition, but what we mean here is the concept of a defined contract between two sections of the system.

```ts
interface AttackAction {
    
    // The name of the attack
    name: string;

    // The power of the attack
    power: number;

    // The type of the attack
    type: AttackType;
    
    // Handlers to handle the attack
    applyAttack(target: Dragon): void;
}
```

## Errors should never be generic

Errors should be specific to the problem they are solving. This allows us to handle errors more effectively and less flakey.

```ts

// We can define a base error class that all other errors extend from which accepts a cause error for routing and tracing

export class BaseError extends Error {
    constructor(message: string, error?: Error) {
        super(message, { cause: error });
    }
}

// Simple extension of a base error class is sufficient for most cases

export class DragonError extends BaseError {}
export class DragonTooYoungError extends BaseError {}
export class DragonAlreadyDeadError extends BaseError {}


```

Since we use errors this way, catch statements can be more specific and handle errors more effectively.

```ts
    try {
        // ...
    } catch (error) {
        if (error instanceof DragonError) {
            // ...
        }
        if (error instanceof DragonTooYoungError) {
            // ...
        }
        if (error instanceof DragonAlreadyDeadError) {
            // ...
        }
        // ...
    }
```

## Tests as specifications

When we write tests, there is a very specific structure we want to follow. Imagine first we have a natural language description of the behaviour of the system.
That could look like this:

```md

### Dragons:
- When we have a dragon which is in a residence 
    - It can try to have a baby
        - Success: The dragon will have a baby
        - Failure: If the dragon is too young, it will throw a DragonTooYoungError
        - Failure: If the dragon is already dead, it will throw a DragonAlreadyDeadError
    - It can try to change its residence
        - Success: The dragon will change its residence
        - Failure: If the new residence is the same as the current residence, it will throw a DragonCannotMoveToSameResidenceError
    - The dragon can be killed
        - Success: The dragon will be killed
        - Failure: If the dragon is already dead, it will throw a DragonAlreadyDeadError
    - The dragon can be revived
        - Success: The dragon will be revived
        - Failure: If the dragon is not dead, it will throw a DragonNotDeadError
- When we have a dragon which is not in a residence
    - It can try to have a baby
        - Failure: If the dragon is too young, it will throw a DragonTooYoungError
        - Failure: If the dragon is already dead, it will throw a DragonAlreadyDeadError
        - Failure: Since the dragon is not in a residence, it will throw a DragonNotInResidenceError
    - It can try to change its residence
        - Success: The dragon will change its residence

```

From this natural language description, we can write tests that are specific to the behaviour of the system. Every leaf node in the above tree should be a "test" expression and every non-leaf node should be a "describe" expression. Here is a partial mapping of the above to tests in vitest, notice how the classes and error types that we were careful to build all come together to form a complete test suite:

```ts

import { describe, test, expect } from 'vitest';

describe('Dragons', () => {
    describe('When we have a dragon which is in a residence', () => {
        describe('It can try to have a baby', () => {
            test('Success: The dragon will have a baby', () => {
                const dragonInResidence = new Dragon({
                    name: 'Dragon',
                    residence: new Castle('Castle'),
                    age: 100,
                });

                const babyDragon = dragonInResidence.haveBaby('Baby Dragon');

                expect(babyDragon).toBeDefined();
                expect(babyDragon.name).toBe('Baby Dragon');
                expect(babyDragon.residence).toBe(dragonInResidence.residence);
                expect(babyDragon.age).toBe(0);
                expect(babyDragon.isDead).toBe(false);
            });
            test('Failure: If the dragon is too young, it will throw a DragonTooYoungError', () => {
                const dragonInResidence = new Dragon({
                    name: 'Dragon',
                    residence: new Castle('Castle'),
                    age: 99,
                });
                expect(() => dragonInResidence.haveBaby('Baby Dragon')).toThrow(DragonTooYoungError);
            });
            test('Failure: If the dragon is already dead, it will throw a DragonAlreadyDeadError', () => {
                const dragonInResidence = new Dragon({
                    name: 'Dragon',
                    residence: new Castle('Castle'),
                    isDead: true,
                });
                expect(() => dragonInResidence.haveBaby('Baby Dragon')).toThrow(DragonAlreadyDeadError);
            });
        });
        
    });
});
```