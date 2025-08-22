# GemenXFreemium_V3
Token Gaming 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title GemenXFreemium_V3
 * @dev ERC20 token with blacklist, burn, owner-controlled mint, reentrancy protection, and ERC20 rescue.
 * OpenZeppelin v4.9.3 imports via GitHub for Remix compilation.
 */

import "https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v4.9.3/contracts/token/ERC20/ERC20.sol";
import "https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v4.9.3/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v4.9.3/contracts/access/Ownable.sol";
import "https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v4.9.3/contracts/security/ReentrancyGuard.sol";
import "https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v4.9.3/contracts/token/ERC20/utils/SafeERC20.sol";
import "https://raw.githubusercontent.com/OpenZeppelin/openzeppelin-contracts/v4.9.3/contracts/token/ERC20/IERC20.sol";

contract GemenXFreemium_V3 is ERC20, ERC20Burnable, Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // Events
    event Mint(address indexed to, uint256 amount);
    event Burned(address indexed from, uint256 amount);
    event Blacklisted(address indexed account);
    event UnBlacklisted(address indexed account);
    event RescueERC20(address indexed token, address indexed to, uint256 amount);

    // Distribution addresses
    address public constant AIRDROP          = 0x4A0f00fC8608Eb30962532F6500C4aF6160bDaDc;
    address public constant LIQUIDITY        = 0x44474050ecDf2Dc3dE0dE9a343d205e87BfCbADb;
    address public constant TEAM_VESTING     = 0x9920c63E3b93A1322ACf2c8DdbE66A9607F5F501;
    address public constant GAMING           = 0xD7d2A6DAe8Cd531Fc8eED9F7E7B5B72cbfBb7eFD;
    address public constant SPONSORS         = 0x39B1DECa70f9b0e47AeF1D1154091771b27f9dF0;
    address public constant RESERVE_WALLET   = 0xe25872B301dA6dac19b3539DA821Ae9d97469d9a;

    // Total supply (500,000,000 tokens, 18 decimals)
    uint256 public constant INITIAL_SUPPLY = 500_000_000 * 10 ** 18;

    // Blacklist mapping
    mapping(address => bool) private _blacklisted;

    // Burn rate in basis points (default 0 = no automatic burn)
    uint256 public burnBps = 0;
    uint256 public constant BPS_DENOM = 10_000;

    constructor() ERC20("GemenXFreemium_V3", "GXF3") {
        // Initial distribution
        _mint(AIRDROP,        (INITIAL_SUPPLY * 25) / 100);
        _mint(LIQUIDITY,      (INITIAL_SUPPLY * 20) / 100);
        _mint(TEAM_VESTING,   (INITIAL_SUPPLY * 15) / 100);
        _mint(GAMING,         (INITIAL_SUPPLY * 25) / 100);
        _mint(SPONSORS,       (INITIAL_SUPPLY * 5)  / 100);
        _mint(RESERVE_WALLET, (INITIAL_SUPPLY * 10) / 100);

        // Sanity check
        require(totalSupply() == INITIAL_SUPPLY, "GXF3: bad initial distribution");
    }

    // ---------------------------
    // Blacklist management
    // ---------------------------
    function blacklist(address account) external onlyOwner {
        require(account != address(0), "GXF3: zero address");
        require(!_blacklisted[account], "GXF3: already blacklisted");
        _blacklisted[account] = true;
        emit Blacklisted(account);
    }

    function unBlacklist(address account) external onlyOwner {
        require(account != address(0), "GXF3: zero address");
        require(_blacklisted[account], "GXF3: not blacklisted");
        _blacklisted[account] = false;
        emit UnBlacklisted(account);
    }

    function isBlacklisted(address account) external view returns (bool) {
        return _blacklisted[account];
    }

    // ---------------------------
    // Burn rate setter
    // ---------------------------
    function setBurnBps(uint256 newBps) external onlyOwner {
        require(newBps <= 500, "GXF3: max 5%");
        burnBps = newBps;
    }

    // ---------------------------
    // Mint (owner only)
    // ---------------------------
    function mint(address to, uint256 amount) external onlyOwner nonReentrant {
        require(to != address(0), "GXF3: zero address");
        require(amount > 0, "GXF3: zero amount");
        require(!_blacklisted[to], "GXF3: recipient blacklisted");
        _mint(to, amount);
        emit Mint(to, amount);
    }

    // ---------------------------
    // Burn (user)
    // ---------------------------
    function burn(uint256 amount) public override nonReentrant {
        require(amount > 0, "GXF3: zero amount");
        require(!_blacklisted[_msgSender()], "GXF3: caller blacklisted");
        super.burn(amount);
        emit Burned(_msgSender(), amount);
    }

    // ---------------------------
    // Transfer overrides
    // ---------------------------
    function _transfer(address from, address to, uint256 amount) internal override {
        require(from != address(0) && to != address(0), "GXF3: zero address");
        require(amount > 0, "GXF3: zero amount");
        require(!_blacklisted[from], "GXF3: sender blacklisted");
        require(!_blacklisted[to], "GXF3: recipient blacklisted");

        // Burn if applicable (except owner)
        if (burnBps > 0 && from != owner() && to != owner()) {
            uint256 fee = (amount * burnBps) / BPS_DENOM;
            if (fee > 0) {
                uint256 remain = amount - fee;
                super._burn(from, fee);
                super._transfer(from, to, remain);
                return;
            }
        }

        super._transfer(from, to, amount);
    }

    function transfer(address to, uint256 amount) public override returns (bool) {
        require(!_blacklisted[_msgSender()], "GXF3: sender blacklisted");
        require(!_blacklisted[to], "GXF3: recipient blacklisted");
        return super.transfer(to, amount);
    }

    function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
        require(!_blacklisted[_msgSender()], "GXF3: caller blacklisted");
        require(!_blacklisted[from], "GXF3: from blacklisted");
        require(!_blacklisted[to], "GXF3: to blacklisted");
        return super.transferFrom(from, to, amount);
    }

    function approve(address spender, uint256 amount) public override returns (bool) {
        require(!_blacklisted[_msgSender()], "GXF3: owner blacklisted");
        require(!_blacklisted[spender], "GXF3: spender blacklisted");
        return super.approve(spender, amount);
    }

    // ---------------------------
    // Rescue ERC20 tokens
    // ---------------------------
    function rescueERC20(address token, address to, uint256 amount) external onlyOwner {
        require(token != address(this), "GXF3: cannot rescue GXF3");
        require(to != address(0), "GXF3: zero address");
        IERC20(token).safeTransfer(to, amount);
        emit RescueERC20(token, to, amount);
    }
}
