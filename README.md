library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;

entity traffic_controller is
    Port (
        clk             : in  STD_LOGIC; -- System clock (e.g., 100MHz)
        rst             : in  STD_LOGIC; -- Reset button ('1' to reset)
        emergency_req   : in  STD_LOGIC; -- Emergency switch ('1' for emergency)

        -- Outputs for LEDs [2: Red, 1: Yellow, 0: Green]
        ns_lights       : out STD_LOGIC_VECTOR(2 downto 0); -- North-South lights
        ew_lights       : out STD_LOGIC_VECTOR(2 downto 0)  -- East-West lights
    );
end traffic_controller;

architecture Behavioral of traffic_controller is

    -- Define the states for the traffic light FSM (Finite State Machine)
    type state_t is (S_NS_GREEN, S_NS_YELLOW, S_EW_GREEN, S_EW_YELLOW, S_EMERGENCY);
    signal current_state : state_t := S_NS_GREEN;

    -- Constants for timer duration. Modify these to change light timings.
    -- This example assumes a 100MHz clock.
    -- For 5 seconds: 5 * 100,000,000 = 500,000,000
    constant GREEN_TIME  : integer := 500_000_000; -- 5 seconds
    constant YELLOW_TIME : integer := 200_000_000; -- 2 seconds

    -- Signal for the timer
    signal timer_counter : integer range 0 to GREEN_TIME := 0;

begin

    -- FSM and Timer Process
    -- This process controls state changes and timing
    process(clk, rst)
    begin
        if rst = '1' then
            -- On reset, go to the default state and reset the timer
            current_state <= S_NS_GREEN;
            timer_counter <= 0;
        elsif rising_edge(clk) then
            -- Check for emergency override first, as it has the highest priority
            if emergency_req = '1' then
                current_state <= S_EMERGENCY;
                timer_counter <= 0;
            else
                -- Check if the timer for the current state has finished
                if timer_counter >= (
                    case current_state is
                        when S_NS_GREEN | S_EW_GREEN => GREEN_TIME,
                        when others => YELLOW_TIME
                    end case
                ) then
                    -- Timer finished, so reset it and move to the next state
                    timer_counter <= 0;
                    case current_state is
                        when S_NS_GREEN  => current_state <= S_NS_YELLOW;
                        when S_NS_YELLOW => current_state <= S_EW_GREEN;
                        when S_EW_GREEN  => current_state <= S_EW_YELLOW;
                        when S_EW_YELLOW => current_state <= S_NS_GREEN;
                        when S_EMERGENCY => -- When emergency is over, return to a safe state
                            if emergency_req = '0' then
                                current_state <= S_NS_GREEN;
                            end if;
                    end case;
                else
                    -- Timer is not finished, so keep counting
                    timer_counter <= timer_counter + 1;
                end if;
            end if;
        end if;
    end process;


    -- Output Logic Process
    -- This process sets the light colors based on the current state
    process(current_state)
    begin
        -- Set default values for lights
        ns_lights <= "100"; -- Red
        ew_lights <= "100"; -- Red

        case current_state is
            when S_NS_GREEN =>
                ns_lights <= "001"; -- North-South is Green
                ew_lights <= "100"; -- East-West is Red
            when S_NS_YELLOW =>
                ns_lights <= "010"; -- North-South is Yellow
                ew_lights <= "100"; -- East-West is Red
            when S_EW_GREEN =>
                ns_lights <= "100"; -- North-South is Red
                ew_lights <= "001"; -- East-West is Green
            when S_EW_YELLOW =>
                ns_lights <= "100"; -- North-South is Red
                ew_lights <= "010"; -- East-West is Yellow
            when S_EMERGENCY =>
                ns_lights <= "100"; -- All lights Red
                ew_lights <= "100"; -- All lights Red
        end case;
    end process;

end Behavioral;
