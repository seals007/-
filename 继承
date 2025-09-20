// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title Inheritance - A smart contract for blockchain-based inheritance
/// @notice Allows owner to set beneficiaries and distribute assets (ETH or ERC20) upon death or timeout
contract Inheritance is Ownable, ReentrancyGuard {
    // Struct to store beneficiary details
    struct Beneficiary {
        address recipient;
        uint256 share; // Percentage of assets (in basis points, 10000 = 100%)
        bool claimed;
    }

    // Mapping of beneficiaries
    mapping(address => Beneficiary) public beneficiaries;
    address[] public beneficiaryList;

    // Last activity timestamp of the owner
    uint256 public lastActive;
    // Time lock duration (e.g., 1 year = 365 days)
    uint256 public constant TIME_LOCK = 365 days;
    // Flag to indicate if distribution is triggered
    bool public distributionTriggered;
    // Oracle or trusted party address
    address public trustedOracle;

    // Events for transparency
    event BeneficiaryAdded(address indexed beneficiary, uint256 share);
    event BeneficiaryUpdated(address indexed beneficiary, uint256 newShare);
    event BeneficiaryRemoved(address indexed beneficiary);
    event DistributionTriggered(address indexed triggerer, uint256 timestamp);
    event AssetsClaimed(address indexed beneficiary, uint256 ethAmount, address token, uint256 tokenAmount);
    event EmergencyWithdraw(address indexed owner, uint256 ethAmount, address token, uint256 tokenAmount);

    // Errors
    error InvalidBeneficiary();
    error InvalidShare();
    error DistributionAlreadyTriggered();
    error NotTriggered();
    error AlreadyClaimed();
    error TimeLockNotExpired();
    error InvalidOracle();
    error NoBeneficiaries();

    /// @notice Constructor to initialize the owner and oracle
    /// @param _oracle Trusted oracle or third-party address
    constructor(address _oracle) Ownable(msg.sender) {
        if (_oracle == address(0)) revert InvalidOracle();
        trustedOracle = _oracle;
        lastActive = block.timestamp;
    }

    /// @notice Update last active timestamp to prevent premature distribution
    function updateActivity() external onlyOwner {
        lastActive = block.timestamp;
    }

    /// @notice Add or update a beneficiary with their share
    /// @param _beneficiary Address of the beneficiary
    /// @param _share Share in basis points (10000 = 100%)
    function addBeneficiary(address _beneficiary, uint256 _share) external onlyOwner {
        if (_beneficiary == address(0)) revert InvalidBeneficiary();
        if (_share == 0 || _share > 10000) revert InvalidShare();
        if (distributionTriggered) revert DistributionAlreadyTriggered();

        if (beneficiaries[_beneficiary].recipient == address(0)) {
            // New beneficiary
            beneficiaryList.push(_beneficiary);
            beneficiaries[_beneficiary] = Beneficiary(_beneficiary, _share, false);
            emit BeneficiaryAdded(_beneficiary, _share);
        } else {
            // Update existing beneficiary
            beneficiaries[_beneficiary].share = _share;
            emit BeneficiaryUpdated(_beneficiary, _share);
        }

        // Validate total shares <= 100%
        uint256 totalShares = 0;
        for (uint256 i = 0; i < beneficiaryList.length; i++) {
            totalShares += beneficiaries[beneficiaryList[i]].share;
        }
        require(totalShares <= 10000, "Total shares exceed 100%");
    }

    /// @notice Remove a beneficiary
    /// @param _beneficiary Address to remove
    function removeBeneficiary(address _beneficiary) external onlyOwner {
        if (distributionTriggered) revert DistributionAlreadyTriggered();
        if (beneficiaries[_beneficiary].recipient == address(0)) revert InvalidBeneficiary();

        delete beneficiaries[_beneficiary];
        for (uint256 i = 0; i < beneficiaryList.length; i++) {
            if (beneficiaryList[i] == _beneficiary) {
                beneficiaryList[i] = beneficiaryList[beneficiaryList.length - 1];
                beneficiaryList.pop();
                break;
            }
        }
        emit BeneficiaryRemoved(_beneficiary);
    }

    /// @notice Trigger distribution by oracle or timeout
    function triggerDistribution() external {
        if (msg.sender != trustedOracle && block.timestamp < lastActive + TIME_LOCK) {
            revert TimeLockNotExpired();
        }
        if (distributionTriggered) revert DistributionAlreadyTriggered();
        if (beneficiaryList.length == 0) revert NoBeneficiaries();

        distributionTriggered = true;
        emit DistributionTriggered(msg.sender, block.timestamp);
    }

    /// @notice Claim assets by a beneficiary
    /// @param _token ERC20 token address (address(0) for ETH)
    function claim(address _token) external nonReentrant {
        if (!distributionTriggered) revert NotTriggered();
        Beneficiary storage beneficiary = beneficiaries[msg.sender];
        if (beneficiary.recipient == address(0)) revert InvalidBeneficiary();
        if (beneficiary.claimed) revert AlreadyClaimed();

        beneficiary.claimed = true;
        uint256 share = beneficiary.share;

        // Distribute ETH
        uint256 ethBalance = address(this).balance;
        if (ethBalance > 0) {
            uint256 ethAmount = (ethBalance * share) / 10000;
            (bool success, ) = payable(msg.sender).call{value: ethAmount}("");
            require(success, "ETH transfer failed");
            emit AssetsClaimed(msg.sender, ethAmount, address(0), 0);
        }

        // Distribute ERC20 token
        if (_token != address(0)) {
            IERC20 token = IERC20(_token);
            uint256 tokenBalance = token.balanceOf(address(this));
            if (tokenBalance > 0) {
                uint256 tokenAmount = (tokenBalance * share) / 10000;
                token.transfer(msg.sender, tokenAmount);
                emit AssetsClaimed(msg.sender, 0, _token, tokenAmount);
            }
        }
    }

    /// @notice Emergency withdraw by owner before distribution
    function emergencyWithdraw(address _token) external onlyOwner nonReentrant {
        if (distributionTriggered) revert DistributionAlreadyTriggered();

        // Withdraw ETH
        uint256 ethBalance = address(this).balance;
        if (ethBalance > 0) {
            (bool success, ) = payable(owner()).call{value: ethBalance}("");
            require(success, "ETH withdraw failed");
            emit EmergencyWithdraw(owner(), ethBalance, address(0), 0);
        }

        // Withdraw ERC20 token
        if (_token != address(0)) {
            IERC20 token = IERC20(_token);
            uint256 tokenBalance = token.balanceOf(address(this));
            if (tokenBalance > 0) {
                token.transfer(owner(), tokenBalance);
                emit EmergencyWithdraw(owner(), 0, _token, tokenBalance);
            }
        }
    }

    /// @notice Receive ETH deposits
    receive() external payable {}

    /// @notice Get total beneficiaries
    function getBeneficiaries() external view returns (address[] memory) {
        return beneficiaryList;
    }
}
