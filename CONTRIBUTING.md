# Contributing Guide

Thank you for your interest in contributing to the Corporate Governance FHEVM example!

## Getting Started

### 1. Fork the Repository

```bash
git clone https://github.com/YOUR_USERNAME/CorporateGovernance.git
cd CorporateGovernance
```

### 2. Create a Branch

```bash
git checkout -b feature/your-feature-name
# or
git checkout -b fix/your-bug-fix
```

### 3. Install Dependencies

```bash
npm install
```

---

## Development Workflow

### Setup

```bash
# Copy environment template
cp .env.example .env

# Install all dependencies
npm install

# Verify setup
npm run compile
npm run test
```

### Making Changes

#### For Smart Contracts

1. **Edit contract in `contracts/`**
   ```solidity
   // Add your changes
   // Include JSDoc comments with chapter tags
   ```

2. **Write tests in `test/`**
   ```typescript
   it("✅ should test your feature", async function () {
       // ✅ for success cases
       // ❌ for error cases
   });
   ```

3. **Compile and test**
   ```bash
   npm run compile
   npm run test
   ```

4. **Check coverage**
   ```bash
   npm run coverage
   ```

#### For Documentation

1. **Edit `.md` files in `docs/`**
   - Use clear section headers
   - Include code examples
   - Add chapter tags (e.g., `chapter: encryption`)

2. **Update `docs/SUMMARY.md` if adding files**

3. **Verify links**
   ```bash
   # Manually check markdown links
   ```

---

## Code Standards

### Solidity Code

**Format:**
```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

/**
 * @title Short description
 * @notice Longer explanation
 * @dev Implementation details
 * chapter: concept-name
 */
contract MyContract {
    // Code here
}
```

**Requirements:**
- ✅ Include SPDX license header
- ✅ Use proper visibility (public/internal/private)
- ✅ Document with JSDoc comments
- ✅ Include chapter tags
- ✅ Format with `prettier` (if available)
- ✅ Use consistent naming (camelCase for functions)
- ✅ Include error messages in requires

**Example:**
```solidity
/**
 * @notice Add a new board member
 * @param _member Address of member
 * @param _name Member's name
 * @param _votingPower Member's voting weight
 * chapter: access-control
 */
function addBoardMember(
    address _member,
    string memory _name,
    uint256 _votingPower
) external {
    require(_votingPower > 0, "Voting power must be > 0");
    // Implementation
}
```

### TypeScript Code

**Format:**
```typescript
/**
 * @title Descriptive name
 * @notice What this does
 * chapter: concept-name
 */

describe("Test Suite", function () {
    it("✅ should test success case", async function () {
        // Arrange
        const expected = 5;

        // Act
        const result = await someFunction();

        // Assert
        expect(result).to.equal(expected);
    });

    it("❌ should reject invalid input", async function () {
        await expect(invalidFunction()).to.be.reverted;
    });
});
```

**Requirements:**
- ✅ Use ✅ for success tests, ❌ for error tests
- ✅ Include JSDoc comments
- ✅ Use AAA pattern (Arrange-Act-Assert)
- ✅ Set adequate gas limits for FHE operations
- ✅ Test both positive and negative cases
- ✅ Clear, descriptive test names

### Markdown Documentation

**Format:**
```markdown
# Title - Concept Name

## Overview
Brief explanation of the concept

## Key Concept
Explain what and why

## Implementation
Show how it works with examples

## Common Mistakes
Show what NOT to do

## Best Practices
Show what TO do

---

**Chapter**: concept-name
**Difficulty**: Beginner/Intermediate/Advanced
**Prerequisites**: Concepts needed first
```

**Requirements:**
- ✅ Include chapter tags at bottom
- ✅ Use code blocks with language (```solidity, ```typescript)
- ✅ Include both good and bad examples
- ✅ Clear section headers
- ✅ Practical examples
- ✅ Links to related concepts

---

## Testing Requirements

### Test Coverage

**Minimum Requirements:**
- ✅ All public functions tested
- ✅ Both success and error cases
- ✅ Edge cases covered
- ✅ Access control verified
- ✅ Gas costs analyzed

**Running Tests:**
```bash
# All tests
npm run test

# Specific test file
npx hardhat test test/CorporateGovernance.ts

# Specific test
npx hardhat test --grep "should cast encrypted vote"

# With coverage
npm run coverage
```

### Test Structure

```typescript
describe("Feature Category", function () {
    // Setup
    beforeEach(async function () {
        // Initialize contract, accounts, etc.
    });

    // Positive cases
    it("✅ should [expected behavior]", async function () {
        // Test code
    });

    // Negative cases
    it("❌ should reject [invalid input]", async function () {
        await expect(promise).to.be.reverted;
    });
});
```

---

## Git Workflow

### Commit Messages

**Format:**
```
Type: Brief description

Longer explanation if needed
- Bullet point
- Bullet point

Closes #123 (if applicable)
```

**Types:**
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation
- `test:` - Test additions
- `refactor:` - Code refactoring
- `style:` - Formatting
- `chore:` - Dependencies, tooling

**Examples:**
```
feat: Add support for weighted voting
- Implement voting power system
- Update test suite
- Add documentation

fix: Resolve gas calculation issue
- Fix TFHE.select gas estimation
- Add clarifying comment

docs: Add gateway decryption guide
```

### Before Submitting PR

```bash
# 1. Update local repo
git fetch origin
git rebase origin/main

# 2. Verify tests pass
npm run test

# 3. Check code quality
npm run lint

# 4. Verify coverage
npm run coverage

# 5. Create commits
git add .
git commit -m "Type: Description"

# 6. Push to your fork
git push origin feature/your-feature
```

---

## Pull Request Process

### 1. Create Pull Request

**Title Format:**
```
[TYPE] Brief description

Examples:
[FEATURE] Add multi-signature support
[FIX] Resolve timestamp validation bug
[DOCS] Update API reference
[TEST] Add edge case tests
```

**Description Template:**
```markdown
## Description
Brief explanation of changes

## Related Issues
Closes #123

## Changes Made
- Change 1
- Change 2
- Change 3

## Testing
- [ ] Added tests
- [ ] All tests pass
- [ ] Coverage maintained

## Documentation
- [ ] Updated README
- [ ] Updated API docs
- [ ] Added chapter tags
```

### 2. Code Review

**Reviewers will check:**
- ✅ Code quality and standards
- ✅ Test coverage
- ✅ Documentation completeness
- ✅ Security considerations
- ✅ Gas optimization
- ✅ FHEVM best practices

**Feedback:**
- Be respectful and constructive
- Explain reasoning for changes
- Link to relevant documentation
- Suggest improvements

### 3. Merge

Once approved:
- Squash commits if needed
- Ensure CI/CD passes
- Merge to main branch

---

## Contribution Ideas

### Code Contributions

- ✅ Add new contract variants
- ✅ Improve existing functions
- ✅ Add gas optimizations
- ✅ Implement security enhancements
- ✅ Add new test cases

### Documentation

- ✅ Clarify existing docs
- ✅ Add new guides
- ✅ Create tutorials
- ✅ Fix examples
- ✅ Add troubleshooting entries

### Community

- ✅ Report bugs
- ✅ Suggest features
- ✅ Write blog posts
- ✅ Create educational content
- ✅ Help other contributors

---

## Development Tips

### Common Commands

```bash
# Compile contracts
npm run compile

# Run tests
npm run test

# Check gas
REPORT_GAS=true npm run test

# Generate coverage
npm run coverage

# Clean artifacts
npm run clean

# View coverage
open coverage/index.html  # macOS
# or
start coverage/index.html  # Windows
```

### Debugging

```typescript
// Add logging
console.log("Debug value:", someValue);

// Check contract state
const state = await contract.getTotalVotingPower();
console.log("State:", state);

// Use Hardhat console
npx hardhat console --network localhost
// In console:
> const c = await ethers.getContractAt("Name", "0x...")
> await c.someFunction()
```

---

## Questions?

- **Setup Issues**: Check TROUBLESHOOTING.md
- **Code Questions**: Review API_REFERENCE.md
- **Testing Help**: See TEST_GUIDE.md
- **Deployment**: Check DEPLOYMENT.md

---

## Recognition

Contributors will be recognized in:
- CONTRIBUTORS.md (when created)
- Project README
- Releases notes

Thank you for making this better! 🙏

---

**Last Updated**: December 2025
**Status**: Welcome contributions!
