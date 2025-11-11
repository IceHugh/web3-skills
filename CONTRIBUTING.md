# Contributing to Web3 Skills

Thank you for your interest in contributing to Web3 Skills! This document provides guidelines and information for contributors.

## 🤝 How to Contribute

### Reporting Bugs

Before creating bug reports, please check the existing issues as you might find that the bug has already been reported. When creating a bug report, please include:

- A clear and descriptive title
- Steps to reproduce the issue
- Expected behavior
- Actual behavior
- Environment information (OS, Node.js version, etc.)
- Any relevant error messages or logs

### Suggesting Features

Feature suggestions are welcome! Please provide:

- A clear description of the feature
- Why this feature would be useful
- How you envision it working
- Any potential implementation ideas

### Code Contributions

#### Setup Development Environment

1. Fork the repository
2. Clone your fork locally
   ```bash
   git clone https://github.com/YOUR_USERNAME/web3-skills.git
   cd web3-skills
   ```

3. Install dependencies
   ```bash
   npm install
   ```

4. Create a new branch
   ```bash
   git checkout -b feature/your-feature-name
   ```

#### Making Changes

1. Follow the existing code style and conventions
2. Write clear, descriptive commit messages
3. Add tests for new functionality
4. Update documentation as needed
5. Ensure all tests pass

#### Running Tests

```bash
# Run all tests
npm test

# Run tests with coverage
npm run test:coverage

# Run linting
npm run lint

# Run security tests
npm run test:security
```

#### Submitting Changes

1. Commit your changes
   ```bash
   git commit -m "feat: add new contract template"
   ```

2. Push to your fork
   ```bash
   git push origin feature/your-feature-name
   ```

3. Create a Pull Request with:
   - Clear title and description
   - Reference to related issues
   - Testing instructions
   - Screenshots if applicable

## 📝 Development Guidelines

### Code Style

We use ESLint and Prettier for code formatting. Please run:

```bash
npm run lint
npm run format
```

### Commit Messages

We follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

- `feat:` - New features
- `fix:` - Bug fixes
- `docs:` - Documentation changes
- `style:` - Code style changes (formatting, etc.)
- `refactor:` - Code refactoring
- `test:` - Adding or updating tests
- `chore:` - Maintenance tasks

### Smart Contract Guidelines

When contributing smart contract templates:

1. **Security First**: Always prioritize security
2. **Use OpenZeppelin**: Leverage battle-tested libraries
3. **Document Everything**: Include comprehensive documentation
4. **Test Thoroughly**: Provide complete test coverage
5. **Gas Optimization**: Consider gas efficiency

#### Contract Template Structure

```
templates/
├── ContractName.sol
├── ContractName.test.js
├── ContractName.md
└── deployment/
    └── deploy-contract-name.js
```

### Documentation

- Update relevant documentation
- Use clear, concise language
- Include code examples
- Follow markdown formatting guidelines

## 🏷️ Areas of Contribution

We particularly welcome contributions in:

### Smart Contract Templates
- Additional ERC standards (ERC777, ERC4626, etc.)
- DeFi protocol templates (AMM, lending, governance)
- NFT marketplace templates
- Cross-chain bridge templates
- Layer 2 specific optimizations

### Security Features
- Enhanced vulnerability detection
- Gas optimization algorithms
- Security audit automation
- Best practices enforcement

### AI Agent Enhancements
- New specialized agents for different domains
- Improved reasoning capabilities
- Better context understanding
- Multi-chain expertise

### Documentation
- Translation to other languages
- Tutorial creation
- Example projects
- API documentation improvements

### Testing
- Additional test scenarios
- Integration tests
- Performance benchmarks
- Security test suites

## 🎯 Pull Request Process

1. **Update Documentation**: Ensure docs are updated
2. **Add Tests**: Include tests for new functionality
3. **Run Tests**: Ensure all tests pass
4. **Update Changelog**: Add entry to CHANGELOG.md
5. **Submit PR**: Create detailed pull request

### PR Review Process

- All PRs require review
- Automated tests must pass
- Code coverage should not decrease
- Documentation must be up to date
- Security review for contract changes

## 📄 Licensing

By contributing to Web3 Skills, you agree that your contributions will be licensed under the MIT License.

## 🚀 Getting Help

- **Discussions**: [GitHub Discussions](https://github.com/icehugh/web3-skills/discussions)
- **Issues**: [GitHub Issues](https://github.com/icehugh/web3-skills/issues)
- **Email**: icehugh@example.com

## 🏆 Recognition

Contributors will be recognized in:
- README.md contributors section
- Release notes
- Annual community highlights

Thank you for contributing to Web3 Skills! 🎉