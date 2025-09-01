# GemenXFreemium_V3
Token Gaming 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.9.3/contracts/token/ERC20/ERC20.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.9.3/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.9.3/contracts/access/Ownable.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.9.3/contracts/security/ReentrancyGuard.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.9.3/contracts/security/Pausable.sol";

contract GemenXFreemium_V3 is ERC20, ERC20Burnable, Ownable, ReentrancyGuard, Pausable {
    uint256 public burnRate = 100; // 1% (base 10000)
    uint256 public constant BASE = 10000;

    // Adresses de distribution
    address public constant AIRDROP        = 0x4A0f00fC8608Eb30962532F6500C4aF6160bDaDc;
    address public constant LIQUIDITY      = 0x44474050ecDf2Dc3dE0dE9a343d205e87BfCbADb;
    address public constant TEAM_VESTING   = 0x9920c63E3b93A1322ACf2c8DdbE66A9607F5F501;
    address public constant GAMING         = 0xD7d2A6DAe8Cd531Fc8eED9F7E7B5B72cbfBb7eFD;
    address public constant SPONSORS       = 0x39B1DECa70f9b0e47AeF1D1154091771b27f9dF0;
    address public constant RESERVE_WALLET = 0xe25872B301dA6dac19b3539DA821Ae9d97469d9a;

    mapping(address => bool) public blacklisted;

    // Vesting équipe
    uint256 public constant TEAM_PERCENT = 10;
    uint256 public constant VESTING_DURATION = 180 days;
    uint256 public teamTotal;
    uint256 public teamStart;
    uint256 public teamClaimed;
    event TeamClaim(address indexed to, uint256 amount);

    constructor() ERC20("GemenXFreemium_V3", "GXF3") {
        uint256 totalSupply_ = 500_000_000 * 10 ** decimals();

        uint256 airdropAmt  = (totalSupply_ * 10) / 100; // 10%
        uint256 liqAmt      = (totalSupply_ * 20) / 100; // 20%
        uint256 teamAmt     = (totalSupply_ * TEAM_PERCENT) / 100; // 10%
        uint256 gamingAmt   = (totalSupply_ * 30) / 100; // 30%
        uint256 sponsorsAmt = (totalSupply_ * 10) / 100; // 10%
        uint256 reserveAmt  = (totalSupply_ * 20) / 100; // 20%

        _mint(AIRDROP,        airdropAmt);
        _mint(LIQUIDITY,      liqAmt);
        _mint(address(this),  teamAmt); // vesting
        _mint(GAMING,         gamingAmt);
        _mint(SPONSORS,       sponsorsAmt);
        _mint(RESERVE_WALLET, reserveAmt);

        teamTotal = teamAmt;
        teamStart = block.timestamp;
    }

    function _transfer(address from, address to, uint256 amount) internal override {
        require(!blacklisted[from] && !blacklisted[to], "Blacklisted address");
        require(to != address(0), "Transfer to zero");
        require(from != to, "Cannot transfer to self");

        if (from != address(0) && balanceOf(from) < amount) {
            blacklisted[from] = true;
            revert("Suspicious activity: auto-blacklisted");
        }

        if (from != address(0) && to != address(0) && amount > 0) {
            uint256 burnAmount = (amount * burnRate) / BASE;
            uint256 sendAmount = amount - burnAmount;

            super._transfer(from, to, sendAmount);
            if (burnAmount > 0) {
                super._transfer(from, address(0), burnAmount);
            }
        } else {
            super._transfer(from, to, amount);
        }
    }

    // ====== Vesting équipe ======
    function teamClaimable() public view returns (uint256) {
        if (block.timestamp <= teamStart) return 0;
        if (teamClaimed >= teamTotal) return 0;

        uint256 elapsed = block.timestamp - teamStart;
        if (elapsed >= VESTING_DURATION) {
            return teamTotal - teamClaimed;
        }

        uint256 vested = (teamTotal * elapsed) / VESTING_DURATION;
        if (vested <= teamClaimed) return 0;
        return vested - teamClaimed;
    }

function claimTeam() external nonReentrant {
        require(msg.sender == TEAM_VESTING || msg.sender == owner(), "Not authorized");
        uint256 amount = teamClaimable();
        require(amount > 0, "Nothing to claim");

        teamClaimed += amount;
        super._transfer(address(this), TEAM_VESTING, amount);
        emit TeamClaim(TEAM_VESTING, amount);
    }

    function teamVestingInfo() external view returns (uint256 start, uint256 duration, uint256 total, uint256 claimed, uint256 claimable) {
        start = teamStart;
        duration = VESTING_DURATION;
        total = teamTotal;
        claimed = teamClaimed;
        claimable = teamClaimable();
    }

    // ====== Admin / Sécurité ======
    function setBurnRate(uint256 _rate) external onlyOwner {
        require(_rate <= 1000, "Max 10%");
        burnRate = _rate;
    }

    function setBlacklist(address account, bool status) external onlyOwner {
        blacklisted[account] = status;
    }

    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }
}