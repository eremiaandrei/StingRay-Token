# StingRay-Token
Smart contract and public resources for STRAY (StingRay Token)
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "hardhat/console.sol"; // Import for logging during tests
import "./lib/ERC20.sol";
import "./lib/Ownable.sol";
import "./lib/ReentrancyGuard.sol";

/**
 * @title StingRay - ERC20 Token with Tax, Timelock, Staking (V3 Ready)
 * @dev Combined features including tax, timelock, staking. Liquidity management is handled externally.
 * @dev This version incorporates feedback regarding charity wallet exclusion, timelock clarity,
 * @dev and code comments for better understanding.
 */
contract StingRay is ERC20, Ownable, ReentrancyGuard {
    // **Constants**
    uint256 public constant MAX_SUPPLY = 21_000_000 * 10 ** 18; // Adjusted to Bitcoin-like supply
    uint256 public constant INITIAL_RELEASE_PERCENTAGE = 5; // 5% initially released
    uint256 public constant RELEASE_PERIOD = 365 days; // 1 year between releases
    uint256 public constant HALVING_PERIOD = 4; // Every 4 scheduled releases, the release amount halves

    // **Tax Configuration**
    uint256 public taxPercentage = 5; // Updated: 5% total tax
    uint256 public liquidityFee = 1; // Updated: 1%
    uint256 public burnFee = 1; // Updated: 1%
    uint256 public marketingFee = 1; // Updated: 1%
    uint256 public charityFee = 1; // Renamed from reflectionFee
    uint256 public stakingRewardFee = 1; // Updated: 1%

    // **Addresses**
    address public marketingWallet;
    address public charityWallet; // Added charity wallet address
    address public constant burnWallet = 0x000000000000000000000000000000000000dEaD;
    // Keep token addresses for reference if needed externally
    address public constant WPOL = 0x0d500B1d8E8eF31E21C99d1Db9A6444d3ADf1270; // WMATIC on Polygon PoS
    address public constant USDC = 0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359; // USDC on Polygon PoS

    // **DEX Integration Variables**
    // Liquidity management is now handled externally via scripts/bots.
    // The contract accumulates the liquidity fee portion of the tax.
    uint256 public accumulatedLiquidityTokens; // Stores tokens collected from liquidityFee

    // **Limits**

    // **Timelock Variables**
    uint256 public totalReleased; // Total tokens released via initial mint + scheduled releases + timelock deactivation
    uint256 public lastReleaseTime; // Timestamp of the last scheduled release
    uint256 public completedReleaseCount; // Number of scheduled releases that have been executed (0-based index)
    uint256 public currentReleasePercentage; // Percentage used for the *next* scheduled release

    // **State Variables**
    bool public tradingEnabled = false;
    bool public timelockActive = true;

    // **Mappings**
    mapping(address => bool) private _isExcludedFromFee;

    // **Staking Variables**
    mapping(address => uint256) private _stakingBalances;
    // mapping(address => uint256) private _stakingTimestamps; // Removed: Timestamp not used for current reward logic
    uint256 public totalStaked;
    uint256 public stakingRewardPool; // Tokens available for distribution to stakers

    // **Pull-Based Rewards System**
    uint256 public rewardPerTokenAccumulated; // Accumulated rewards per token staked (scaled by 1e18)
    mapping(address => uint256) public rewardsClaimed; // Total rewards earned and claimed by each user (scaled by 1e18)

    // **Governance Variables**
    bool public ownerRenounced = false;

    // **Events**
    event TaxUpdated(uint256 taxPercentage, uint256 liquidityFee, uint256 burnFee, uint256 marketingFee, uint256 charityFee, uint256 stakingRewardFee);
    event TradingEnabled();
    event MarketingWalletUpdated(address marketingWallet); // For backward compatibility
    event CharityWalletUpdated(address charityWallet); // For backward compatibility
    event Staked(address indexed user, uint256 amount);
    event Unstaked(address indexed user, uint256 amount);
    event RewardsDistributed(uint256 amount);
    event RewardsClaimed(address indexed user, uint256 amount);
    event TokensReleased(uint256 amount, uint256 releaseNumber, uint256 timestamp); // releaseNumber is the completed count (1, 2, 3...)
    event OwnershipRenounced(uint256 timestamp);
    event TimelockDeactivated(uint256 timestamp);
    event StakingPoolFunded(address indexed funder, uint256 amount);
    event LiquidityTokensWithdrawn(uint256 amount);

    // **Custom Errors**
    error InvalidMarketingWallet();
    error InvalidCharityWallet();
    error FeesDoNotSumToTaxPercentage();
    error AmountMustBePositive();
    error InsufficientBalance(uint256 currentBalance, uint256 requiredAmount);
    error InsufficientStakedBalance(uint256 currentStaked, uint256 requiredAmount);
    error InsufficientRewardPool();
    error NoTokensStaked();
    error TradingNotEnabled();
    error TimelockNotReady(uint256 currentTime, uint256 nextReleaseTime);
    error TimelockAlreadyDeactivated();
    error OwnershipAlreadyRenounced();
    error NotEnoughTokensToRelease();
    error InsufficientLiquidityTokens(uint256 available, uint256 requested);

    /**
     * @notice Constructs the StingRay token contract.
     * @param _marketingWallet The address for marketing fee distribution.
     * @param _charityWallet The address for charity fee distribution.
     */
    constructor(address _marketingWallet, address _charityWallet) ERC20("StingRay", "STRAY") Ownable(msg.sender) {
        if (_marketingWallet == address(0)) revert InvalidMarketingWallet();
        if (_charityWallet == address(0)) revert InvalidCharityWallet();

        marketingWallet = _marketingWallet;
        charityWallet = _charityWallet;

        // Calculate the initial release amount (5% of MAX_SUPPLY)
        uint256 initialReleaseAmount = (MAX_SUPPLY * INITIAL_RELEASE_PERCENTAGE) / 100;

        // Mint initial release tokens to deployer
        _mint(msg.sender, initialReleaseAmount);

        // Store the rest of the tokens in the contract for timed release
        _mint(address(this), MAX_SUPPLY - initialReleaseAmount);

        // Set initial timelock variables
        totalReleased = initialReleaseAmount;
        lastReleaseTime = block.timestamp;
        completedReleaseCount = 0; // 0 scheduled releases completed so far
        currentReleasePercentage = INITIAL_RELEASE_PERCENTAGE; // This is the percentage for the *next* release

        // Exclude from fees: deployer, contract itself, burn, marketing, and **charity** wallets.
        _isExcludedFromFee[msg.sender] = true;
        _isExcludedFromFee[address(this)] = true;
        _isExcludedFromFee[burnWallet] = true;
        _isExcludedFromFee[marketingWallet] = true;
        _isExcludedFromFee[charityWallet] = true; // **FIX: Exclude charity wallet from fees**

        // Initialize with trading disabled
        tradingEnabled = false;
    }

    /**
     * @dev Internal function that performs token transfers.
     * @param sender The address sending tokens.
     * @param recipient The address receiving tokens.
     * @param amount The amount of tokens being transferred.
     */
    function _update(address sender, address recipient, uint256 amount) internal override {
        // Bypass all checks and tax logic for excluded addresses or during minting (sender == address(0))
        if (sender == address(0) || _isExcludedFromFee[sender] || _isExcludedFromFee[recipient]) {
            super._update(sender, recipient, amount);
            return;
        }

        // Trading enabled check for non-excluded senders
        if (!tradingEnabled) {
            revert TradingNotEnabled();
        }

        // Whale protection checks removed

        // Calculate and apply tax for transfers between non-excluded accounts
        uint256 taxAmount = (amount * taxPercentage) / 100;
        uint256 transferAmount = amount - taxAmount;

        if (taxAmount > 0) {
            // Calculate individual fee amounts proportional to the tax percentage
            uint256 burnAmount = (taxAmount * burnFee) / taxPercentage;
            uint256 liquidityAmount = (taxAmount * liquidityFee) / taxPercentage;
            uint256 marketingAmount = (taxAmount * marketingFee) / taxPercentage;
            uint256 charityAmount = (taxAmount * charityFee) / taxPercentage;
            uint256 stakingAmount = (taxAmount * stakingRewardFee) / taxPercentage;

            // --- Process Fee Transfers ---
            // Burn fee: transfer to burn wallet
            if (burnAmount > 0) {
                super._update(sender, burnWallet, burnAmount);
            }
            // Marketing fee: transfer to marketing wallet (excluded from fee again upon arrival)
            if (marketingAmount > 0) {
                super._update(sender, marketingWallet, marketingAmount);
            }
            // Liquidity fee: transfer to this contract, increment accumulated balance
            if (liquidityAmount > 0) {
                super._update(sender, address(this), liquidityAmount);
                accumulatedLiquidityTokens += liquidityAmount;
            }
            // Charity fee: transfer to charity wallet (excluded from fee again upon arrival)
            if (charityAmount > 0 && charityWallet != address(0)) {
                super._update(sender, charityWallet, charityAmount);
            }
            // Staking fee: transfer to this contract, add to reward pool
            if (stakingAmount > 0) {
                super._update(sender, address(this), stakingAmount);
                stakingRewardPool += stakingAmount;
            }
            // --- End Fee Transfers ---

            // Process the primary transfer of the remaining amount
            if (transferAmount > 0) {
                 super._update(sender, recipient, transferAmount);
            }
        } else {
            // No tax, perform a normal transfer
            super._update(sender, recipient, amount);
        }
    }

    // **Staking Functions**

    /**
     * @notice Stakes tokens to earn rewards.
     * @param amount The amount of tokens to stake.
     */
    function stake(uint256 amount) external nonReentrant {
        if (amount == 0) revert AmountMustBePositive();
        uint256 senderBalance = balanceOf(msg.sender);
        if (senderBalance < amount) revert InsufficientBalance(senderBalance, amount);

        _claimRewardsInternal(); // Claim any pending rewards before changing stake amount

        _update(msg.sender, address(this), amount); // Transfer tokens to contract
        _stakingBalances[msg.sender] += amount;
        // _stakingTimestamps[msg.sender] = block.timestamp; // Timestamp not currently used for reward calculation
        totalStaked += amount;

        emit Staked(msg.sender, amount);
    }

    /**
     * @notice Unstakes tokens.
     * @param amount The amount of tokens to unstake.
     */
    function unstake(uint256 amount) external nonReentrant {
        if (amount == 0) revert AmountMustBePositive();
        uint256 currentStaked = _stakingBalances[msg.sender];
        if (currentStaked < amount) revert InsufficientStakedBalance(currentStaked, amount);

        _claimRewardsInternal(); // Claim any pending rewards before changing stake amount

        _stakingBalances[msg.sender] -= amount;
        // _stakingTimestamps[msg.sender] = block.timestamp; // Timestamp not currently used for reward calculation
        totalStaked -= amount;
        _update(address(this), msg.sender, amount); // Transfer tokens back to user

        emit Unstaked(msg.sender, amount);
    }

    // **Pull-Based Rewards System**

    /**
     * @notice Distributes tokens from the staking reward pool to stakers.
     * @dev Can only be called by the owner (likely a timelock contract).
     */
    function distributeStakingRewards() external onlyOwner {
        if (stakingRewardPool == 0) revert InsufficientRewardPool();
        if (totalStaked == 0) revert NoTokensStaked(); // Prevent division by zero

        uint256 amountToDistribute = stakingRewardPool;

        // Update the reward per token value for future claims
        rewardPerTokenAccumulated += (amountToDistribute * 1e18) / totalStaked;

        stakingRewardPool = 0; // Empty the pool after distribution

        emit RewardsDistributed(amountToDistribute);
    }

    /**
     * @notice Allows a user to claim their accumulated staking rewards.
     */
    function claimRewards() external nonReentrant {
        _claimRewardsInternal();
    }

    /**
     * @dev Internal function to calculate and transfer unclaimed rewards to the caller.
     */
    function _claimRewardsInternal() private {
        uint256 stakedAmount = _stakingBalances[msg.sender];
        if (stakedAmount == 0) return; // No rewards if not staked

        // Calculate the total rewards earned based on current rewardPerTokenAccumulated
        uint256 totalEarned = (stakedAmount * rewardPerTokenAccumulated) / 1e18;

        // Get the amount already claimed by this user (scaled)
        uint256 claimed = rewardsClaimed[msg.sender];

        // Calculate unclaimed rewards (scaled)
        // Use a safe subtraction pattern
        uint256 unclaimedRewardScaled;
        if (totalEarned > claimed) {
            unclaimedRewardScaled = totalEarned - claimed;
        } else {
            return; // No new rewards to claim
        }

        // The actual token amount to transfer
        uint256 unclaimedReward = unclaimedRewardScaled; // No need to scale back as claimed[user] stores scaled value

        if (unclaimedReward > 0) {
            // Update the user's claimed amount *before* the transfer
            rewardsClaimed[msg.sender] = totalEarned;

            // Check if contract has enough balance (should have from stakingRewardPool additions)
             uint256 contractBalance = balanceOf(address(this));
             if (contractBalance < unclaimedReward) {
                  // This indicates an issue with the accounting or funding.
                  // For safety, just return without transferring if balance is low.
                  return;
             }

            _update(address(this), msg.sender, unclaimedReward); // Transfer rewards
            emit RewardsClaimed(msg.sender, unclaimedReward);
        }
    }

    /**
     * @notice Allows the owner (Timelock) to add funds directly to the staking reward pool.
     * @dev Tokens are transferred from the owner's balance to the contract's reward pool.
     * @param amount The amount of tokens to add to the pool.
     */
    function fundStakingPool(uint256 amount) external onlyOwner {
        if (amount == 0) revert AmountMustBePositive();
        // Transfer from owner (msg.sender) to this contract's staking pool
        _update(msg.sender, address(this), amount);
        stakingRewardPool += amount;
        emit StakingPoolFunded(msg.sender, amount);
    }

    // **Timelock Functions**

    /**
     * @notice Executes a scheduled token release from the contract's locked supply.
     * @dev Can only be called by the owner (likely a timelock contract).
     * @dev Releases occur at fixed intervals (RELEASE_PERIOD).
     * @dev The release amount halves every HALVING_PERIOD releases.
     */
    function releaseTokens() external onlyOwner {
        if (!timelockActive) revert TimelockAlreadyDeactivated();

        // Check if enough time has passed since the last release
        uint256 nextReleaseTime = lastReleaseTime + RELEASE_PERIOD;
        if (block.timestamp < nextReleaseTime) {
            revert TimelockNotReady(block.timestamp, nextReleaseTime);
        }

        // The number of the next release (1st, 2nd, 3rd...)
        uint256 nextReleaseNumber = completedReleaseCount + 1;
        
        // Calculate the percentage for THIS release
        uint256 percentageForThisRelease;
        
        // For the 4th release (and multiples of 4), we need to halve the percentage
        if (nextReleaseNumber > 0 && nextReleaseNumber % HALVING_PERIOD == 0) {
            // For the 4th release, use INITIAL_RELEASE_PERCENTAGE / 2
            // For the 8th release, use INITIAL_RELEASE_PERCENTAGE / 4
            // For the 12th release, use INITIAL_RELEASE_PERCENTAGE / 8
            // etc.
            uint256 halvingCount = nextReleaseNumber / HALVING_PERIOD;
            percentageForThisRelease = INITIAL_RELEASE_PERCENTAGE;
            for (uint256 i = 0; i < halvingCount; i++) {
                percentageForThisRelease = percentageForThisRelease / 2;
            }
        } else {
            percentageForThisRelease = currentReleasePercentage;
        }
        
        // Calculate the amount based on the percentage
        uint256 amountToRelease = (MAX_SUPPLY * percentageForThisRelease) / 100;

        // Check if there are any tokens remaining to release from the timelock part
        uint256 remainingSupply = getRemainingSupply();
        if (remainingSupply == 0) revert NotEnoughTokensToRelease();

        // Ensure we don't release more than what's available
        if (amountToRelease > remainingSupply) {
            amountToRelease = remainingSupply;
        }

        // Update timelock variables
        lastReleaseTime = block.timestamp;
        completedReleaseCount++;
        
        // Update currentReleasePercentage for the next release
        if (nextReleaseNumber % HALVING_PERIOD == 0) {
            currentReleasePercentage = percentageForThisRelease;
        }

        // Transfer tokens to the owner
        _update(address(this), owner(), amountToRelease);

        totalReleased += amountToRelease;

        // Emit with the completed release count
        emit TokensReleased(amountToRelease, completedReleaseCount, block.timestamp);
    }

    /**
     * @notice Calculates the amount of tokens that would be released in the next scheduled release.
     * @return The amount of tokens for the next release.
     */
    function getNextReleaseAmount() public view returns (uint256) {
        if (!timelockActive) return 0;

        // The number of the next release (1st, 2nd, 3rd...)
        uint256 nextReleaseNumber = completedReleaseCount + 1;

        // Calculate the percentage for the next release
        uint256 percentageForNextRelease;
        
        // For the 4th release (and multiples of 4), we need to halve the percentage
        if (nextReleaseNumber > 0 && nextReleaseNumber % HALVING_PERIOD == 0) {
            // For the 4th release, use INITIAL_RELEASE_PERCENTAGE / 2
            // For the 8th release, use INITIAL_RELEASE_PERCENTAGE / 4
            // For the 12th release, use INITIAL_RELEASE_PERCENTAGE / 8
            // etc.
            uint256 halvingCount = nextReleaseNumber / HALVING_PERIOD;
            percentageForNextRelease = INITIAL_RELEASE_PERCENTAGE;
            for (uint256 i = 0; i < halvingCount; i++) {
                percentageForNextRelease = percentageForNextRelease / 2;
            }
        } else {
            percentageForNextRelease = currentReleasePercentage;
        }

        // Calculate the amount based on the percentage
        uint256 releaseAmount = (MAX_SUPPLY * percentageForNextRelease) / 100;

        // Check against remaining supply
        uint256 remainingSupply = getRemainingSupply();
        if (remainingSupply < releaseAmount) {
            releaseAmount = remainingSupply; // Return whatever is left
        }

        return releaseAmount;
    }


    /**
     * @notice Gets the time remaining until the next scheduled token release.
     * @return The time in seconds until the next release, or 0 if timelock is inactive or release is ready.
     */
    function getTimeUntilNextRelease() external view returns (uint256) {
        if (!timelockActive) return 0; // No next release if deactivated
        uint256 nextReleaseTime = lastReleaseTime + RELEASE_PERIOD;
        if (block.timestamp >= nextReleaseTime) {
            return 0; // Release is ready now or overdue
        }
        return nextReleaseTime - block.timestamp;
    }

    /**
     * @notice Permanently deactivates the timelock and releases all remaining locked tokens to the owner.
     * @dev This is a one-way action and cannot be undone.
     * @dev Can only be called by the owner (likely a timelock contract).
     */
    function deactivateTimelock() external onlyOwner {
        if (!timelockActive) revert TimelockAlreadyDeactivated();

        // Set timelock as inactive
        timelockActive = false;

        // Calculate balance held by the contract that is part of the timelock (not staked or reward pool)
        uint256 contractBalance = balanceOf(address(this));
        uint256 nonTimelockBalance = totalStaked + stakingRewardPool + accumulatedLiquidityTokens; // Include accumulated liquidity tokens
        uint256 remainingTokens = contractBalance > nonTimelockBalance ? contractBalance - nonTimelockBalance : 0;

        if (remainingTokens > 0) {
            // Transfer all remaining locked tokens to the owner
            _update(address(this), owner(), remainingTokens);
            totalReleased += remainingTokens; // Update total released amount
        }
        // Note: completedReleaseCount is not updated as this is an override

        emit TimelockDeactivated(block.timestamp);
    }

    /**
     * @notice Allows the contract owner to permanently renounce ownership.
     * @dev This is a one-way action and cannot be undone. Ownership is transferred to the zero address.
     * @dev The `ownerRenounced` flag prevents accidental renunciation or repeated calls.
     */
    function renounceOwnershipPermanently() external onlyOwner {
        if (ownerRenounced) revert OwnershipAlreadyRenounced();

        ownerRenounced = true; // Set the flag

        // Actually renounce ownership using the inherited Ownable function
        renounceOwnership();

        emit OwnershipRenounced(block.timestamp);
    }

    // **View Functions**

    /**
     * @notice Gets the staked balance for a given account.
     * @param account The address to check.
     * @return The amount of tokens staked by the account.
     */
    function getStakedBalance(address account) external view returns (uint256) {
        return _stakingBalances[account];
    }

    /**
     * @notice Calculates the unclaimed staking rewards for a given account.
     * @param account The address to check.
     * @return The amount of tokens the account can claim as rewards.
     */
    function getUnclaimedRewards(address account) external view returns (uint256) {
        uint256 stakedAmount = _stakingBalances[account];
        if (stakedAmount == 0) return 0;

        // Calculate total earned based on current state (scaled)
        uint256 totalEarned = (stakedAmount * rewardPerTokenAccumulated) / 1e18;

        // Get amount already claimed (scaled)
        uint256 claimed = rewardsClaimed[account];

        // Calculate unclaimed amount (scaled)
        return totalEarned > claimed ? totalEarned - claimed : 0; // Return 0 if totalEarned <= claimed
    }

    /**
     * @notice Checks if an account is excluded from transfer fees and limits.
     * @param account The address to check.
     * @return True if the account is excluded, false otherwise.
     */
    function isExcludedFromFee(address account) external view returns (bool) {
        return _isExcludedFromFee[account];
    }

    /**
     * @notice Calculates the total circulating supply of tokens.
     * @dev This is the total supply minus tokens in the burn wallet and tokens locked in the contract's timelock.
     * @return The circulating supply amount.
     */
    function getTotalCirculatingSupply() external view returns (uint256) {
        uint256 contractTimelockBalance = 0;
        if (timelockActive) {
             uint256 contractBalance = balanceOf(address(this));
             uint256 nonTimelockBalance = totalStaked + stakingRewardPool + accumulatedLiquidityTokens; // Include accumulated liquidity tokens as they are held for external use, not timelock
             // The timelock balance is the contract's balance MINUS tokens staked, in reward pool, or accumulated for liquidity
             contractTimelockBalance = contractBalance > nonTimelockBalance ? contractBalance - nonTimelockBalance : 0;
        }
        // Circulating supply is the MAX_SUPPLY minus tokens burnt and tokens still locked in the timelock part of the contract
        return MAX_SUPPLY - balanceOf(burnWallet) - contractTimelockBalance;
    }

    /**
     * @notice Gets the amount of tokens remaining to be released from the timelock.
     * @dev These are tokens held by the contract that are not staked, in the reward pool, or accumulated liquidity.
     * @return The amount of tokens remaining in the timelock, or 0 if timelock is inactive.
     */
    function getRemainingSupply() public view returns (uint256) {
         if (!timelockActive) return 0;
         uint256 contractBalance = balanceOf(address(this));
         // Non-timelock balance includes staked, reward pool, and accumulated liquidity tokens
         uint256 nonTimelockBalance = totalStaked + stakingRewardPool + accumulatedLiquidityTokens;
         return contractBalance > nonTimelockBalance ? contractBalance - nonTimelockBalance : 0;
    }
    
    /**
     * @notice Returns the current release counter (for backward compatibility)
     * @return The release counter (1-based index)
     */
    function releaseCounter() public view returns (uint256) {
        return completedReleaseCount + 1;
    }

    /**
     * @notice Provides details about the next scheduled token release.
     * @return nextReleaseTimestamp The timestamp of the next release.
     * @return nextReleaseAmount The calculated amount for the next release.
     * @return percentageForNextRelease The percentage rate used for the next release calculation (potentially halved).
     * @return releaseNum The number of the next release (1, 2, 3...).
     * @return isHalvingRelease True if the next release is a halving release.
     */
    function getCurrentReleaseDetails() external view returns (
        uint256 nextReleaseTimestamp,
        uint256 nextReleaseAmount,
        uint256 percentageForNextRelease, // Renamed for clarity
        uint256 releaseNum, // Renamed for clarity
        bool isHalvingRelease
    ) {
        nextReleaseTimestamp = timelockActive ? lastReleaseTime + RELEASE_PERIOD : 0;
        nextReleaseAmount = timelockActive ? getNextReleaseAmount() : 0; // Use helper function

        // Calculate the percentage for the next release
        uint256 nextReleaseNumber = completedReleaseCount + 1;
        
        // For the 4th release (and multiples of 4), we need to halve the percentage
        if (nextReleaseNumber > 0 && nextReleaseNumber % HALVING_PERIOD == 0) {
            // For the 4th release, use INITIAL_RELEASE_PERCENTAGE / 2
            // For the 8th release, use INITIAL_RELEASE_PERCENTAGE / 4
            // For the 12th release, use INITIAL_RELEASE_PERCENTAGE / 8
            // etc.
            uint256 halvingCount = nextReleaseNumber / HALVING_PERIOD;
            percentageForNextRelease = INITIAL_RELEASE_PERCENTAGE;
            for (uint256 i = 0; i < halvingCount; i++) {
                percentageForNextRelease = percentageForNextRelease / 2;
            }
        } else {
            percentageForNextRelease = currentReleasePercentage;
        }

        releaseNum = nextReleaseNumber; // The number of the next release

        // Check if the next release number marks a halving period
        isHalvingRelease = (nextReleaseNumber > 0 && nextReleaseNumber % HALVING_PERIOD == 0);

        return (
            nextReleaseTimestamp,
            nextReleaseAmount,
            percentageForNextRelease,
            releaseNum,
            isHalvingRelease
        );
    }

    // **Basic Functions**

    /**
     * @notice Enables trading for the token.
     * @dev Can only be called by the owner (likely a timelock contract).
     * @dev This is a one-way action and cannot be undone.
     */
    function enableTrading() external onlyOwner {
        tradingEnabled = true;
        emit TradingEnabled();
    }

    /**
     * @notice Updates the tax configuration.
     * @param _taxPercentage The new total tax percentage.
     * @param _liquidityFee The new liquidity fee percentage.
     * @param _burnFee The new burn fee percentage.
     * @param _marketingFee The new marketing fee percentage.
     * @param _charityFee The new charity fee percentage.
     * @param _stakingRewardFee The new staking reward fee percentage.
     */
    function updateTaxes(uint256 _taxPercentage, uint256 _liquidityFee, uint256 _burnFee, uint256 _marketingFee, uint256 _charityFee, uint256 _stakingRewardFee) external onlyOwner {
        uint256 totalFees = _liquidityFee + _burnFee + _marketingFee + _charityFee + _stakingRewardFee;
        require(totalFees == _taxPercentage, "Fees must exactly sum to the total tax percentage");
        require(_taxPercentage <= 100, "Total tax percentage cannot exceed 100");
        require(_taxPercentage > 0, "Tax percentage must be greater than zero");  // Add new check
        taxPercentage = _taxPercentage;
        liquidityFee = _liquidityFee;
        burnFee = _burnFee;
        marketingFee = _marketingFee;
        charityFee = _charityFee;
        stakingRewardFee = _stakingRewardFee;
        emit TaxUpdated(_taxPercentage, _liquidityFee, _burnFee, _marketingFee, _charityFee, _stakingRewardFee);
    }

    /**
     * @notice Excludes or includes an account from fees.
     * @param account The address to exclude or include.
     * @param excluded Whether to exclude the account.
     */
    function excludeFromFee(address account, bool excluded) external onlyOwner {
        _isExcludedFromFee[account] = excluded;
    }


    /**
     * @notice Updates the marketing wallet address.
     * @param _marketingWallet The new marketing wallet address.
     */
    function updateMarketingWallet(address _marketingWallet) external onlyOwner {
        require(_marketingWallet != address(0), "InvalidMarketingWallet");
        marketingWallet = _marketingWallet;
        _isExcludedFromFee[_marketingWallet] = true;  // Ensure the new wallet is excluded from fees
        emit MarketingWalletUpdated(_marketingWallet);
    }

    /**
     * @notice Updates the charity wallet address.
     * @param _charityWallet The new charity wallet address.
     */
    function updateCharityWallet(address _charityWallet) external onlyOwner {
        require(_charityWallet != address(0), "InvalidCharityWallet");
        charityWallet = _charityWallet;
        _isExcludedFromFee[_charityWallet] = true;  // Ensure the new wallet is excluded from fees
        emit CharityWalletUpdated(_charityWallet);
    }

    /**
     * @notice Withdraws accumulated liquidity tokens from the contract's balance.
     * @dev Ensures that the withdrawal amount does not exceed the actual token balance held by the contract
     *      or the tracked accumulated liquidity tokens. Reverts if either balance is insufficient.
     * @param amount The amount of tokens to withdraw.
     */
function withdrawLiquidityTokens(uint256 amount) external onlyOwner nonReentrant { // Added nonReentrant
    if (amount == 0) revert AmountMustBePositive();

    uint256 actualBalance = balanceOf(address(this));

    // Check 1: Does the contract actually hold enough tokens?
    if (actualBalance < amount) {
        revert InsufficientBalance(actualBalance, amount); // Revert if actual balance is too low
    }

    // Check 2: Is the requested amount valid according to the accumulated liquidity accounting?
    if (accumulatedLiquidityTokens < amount) {
        // This check ensures the accounting variable also permits the withdrawal.
        revert InsufficientLiquidityTokens(accumulatedLiquidityTokens, amount);
    }

    // If both checks pass, proceed with the withdrawal
    // Decrement accumulated liquidity tokens *before* transfer
    accumulatedLiquidityTokens -= amount;

    // Transfer the requested amount to the owner using _update
    _update(address(this), owner(), amount);

    emit LiquidityTokensWithdrawn(amount);
}
}
